# Prompt: surface stop-sequence value + thinking signature in erllama

Drop this verbatim into a fresh Claude Code session pointed at the
sibling `erllama` repo (`/Users/benoitc/Projects/erllama`).

---

# Surface stop-sequence value and thinking signature

## Context

The downstream `erllama_server` HTTP front end serves an Anthropic
Messages-compatible `/v1/messages` endpoint. Two pieces of data that
Anthropic's spec requires are not currently surfaced by erllama, so
the server has no way to populate them and the SDK round-trip is
lossy:

1. **`stop_sequence` value on natural stop**. Anthropic's response
   carries `stop_reason: "stop_sequence"` and `stop_sequence: "<value>"`
   when generation halts because one of the caller-supplied
   `stop_sequences` strings was hit. Today erllama reports
   `finish_reason :: stop | length | cancelled` on
   `{erllama_done, Ref, Stats}` but does not distinguish "natural
   end-of-generation" from "user-supplied stop string fired" and never
   reports which string fired. The downstream emits
   `stop_reason: "end_turn"` and `stop_sequence: null` in both cases.

2. **Thinking signature for `signature_delta`**. Anthropic's extended
   thinking emits a per-block `signature_delta` SSE event immediately
   before the matching `content_block_stop` for any thinking block,
   carrying an integrity signature that thinking-capable SDKs verify
   and round-trip on subsequent turns. Today erllama emits
   `{erllama_token, Ref, {thinking_delta, Bin}}` text fragments but
   exposes no signature, so the downstream cannot emit a
   spec-conforming `signature_delta` and thinking-capable SDKs reject
   the assistant turn on the next request.

## Files

- `src/erllama.erl` - public façade. The `infer/4` doc + spec live
  here; document additions go here.
- `src/erllama_model.erl` - gen_statem owning the sampler chain and
  the `stream_emit` path. The stop-sequence detector + signature
  surfacing land here.
- `c_src/erllama_nif.c` - if the underlying llama.cpp sampler chain
  needs to surface anything new (signature). For stop-sequence the
  comparison is done in Erlang post-detokenize so no C is needed.
- `test/erllama_model_tests.erl` (or eunit equivalent) - new cases.

## Ask 1: `stop_sequence` in the done message

### Proposed API

`infer/4`'s `Params` map already accepts `stop_sequences :: [binary()]`
(the downstream passes them in directly). Extend `stream_emit`'s
sampler loop so that after each freshly detokenized chunk it scans the
accumulated generated text for a match against any of the supplied
stop strings. On a hit:

1. Halt generation as today.
2. Set `finish_reason = stop` (unchanged - this stays the generic
   "natural stop").
3. **Add `stop_sequence :: binary()`** to the `Stats` map in
   `{erllama_done, Ref, Stats}`. The value is the binary of the matched
   stop string. When generation ended for any other reason
   (`length`, `cancelled`, or natural end-of-stream without a match),
   `stop_sequence` is absent from the map (not `undefined`,
   not `null` - just absent).

Optional but recommended: also truncate the emitted text deltas so the
matched stop string does not leak into the streamed body. Today the
sampler stops mid-token sometimes; if the match crosses a
detokenization boundary, the downstream sees the match prefix as a
final `{erllama_token, _, Bin}` text fragment. Anthropic strips the
match from `content[].text`; we should too, but this is a quality of
output choice, not a correctness break.

### Acceptance criteria

- `Stats` carries `stop_sequence => <<"END">>` when a request with
  `stop_sequences => [<<"END">>]` produced text containing `"END"`.
- The key is absent when generation hit `length`, was cancelled, or
  reached end-of-generation without a stop-string match.
- Multiple stop strings: the first one to match wins (scan order is
  the caller's list order).
- A stop string that overlaps end-of-generation (model emitted EOS at
  the same step) reports `finish_reason = stop` and the matched
  `stop_sequence` rather than the EOS path.

### Test plan

```erlang
%% stop_sequence value carried on natural stop-string match
stop_sequence_value_reported_test() ->
    {ok, M} = test_load_small_model([]),
    {ok, Result} = erllama:complete(M, <<"hello">>,
        #{stop_sequences => [<<"END">>, <<"STOP">>]}),
    case Result of
        #{finish_reason := stop, stop_sequence := S} ->
            ?assert(lists:member(S, [<<"END">>, <<"STOP">>]));
        #{finish_reason := length} ->
            ok  % model never produced either; test passes vacuously
    end.

%% absent when generation hit response_tokens
stop_sequence_absent_on_length_test() ->
    {ok, M} = test_load_small_model([]),
    {ok, Result} = erllama:complete(M, <<"hello">>,
        #{response_tokens => 1,
          stop_sequences => [<<"unlikely-to-match">>]}),
    ?assertNot(maps:is_key(stop_sequence, Result)).
```

Plus a streaming-side test asserting the same shape on
`{erllama_done, _, Stats}`.

## Ask 2: thinking signature surfacing

### Background

This applies only once erllama grows a thinking-aware sampler. Today
`{thinking_delta, _}` tokens are produced by the stub backend in
tests and (eventually) by a real thinking pipeline. The Anthropic
spec requires an integrity signature on each thinking block, emitted
as a separate `content_block_delta` of type `signature_delta`
immediately before the `content_block_stop` for that thinking block.

### Proposed API

Two options; the engine should pick whichever fits its pipeline:

**Option A - per-block signature in the close marker.** Add a new
message tag:

```erlang
{erllama_thinking_end, Ref, Signature :: binary()}
```

emitted by the model gen_statem when the thinking phase closes and
*before* any non-thinking token is emitted. `Signature` is opaque to
the downstream; it is forwarded verbatim into the SSE
`signature_delta` event. If no signature is available (stub backend,
local-only thinking), emit `Signature = <<>>` and the downstream
will omit `signature_delta` from the stream.

**Option B - signature carried on each thinking_delta.** Extend the
existing message to `{erllama_token, Ref, {thinking_delta, Bin,
PartialSig :: binary()}}`. The downstream buffers `PartialSig` and
emits the final value before `content_block_stop`. Less ergonomic
but maps better to streaming sampler designs where the signature is
produced incrementally.

### Acceptance criteria

- For requests with `thinking = enabled` and a model that produces
  thinking blocks, the engine surfaces either an
  `erllama_thinking_end` message or a final-segment signature
  before any non-thinking token / `erllama_done` arrives.
- For requests with `thinking = disabled` or models without
  thinking support, the new mechanism is dormant: no new messages
  and no shape change to existing ones.
- The stub backend in tests can emit a synthetic signature
  (e.g. `sha256(generated_thinking_text)`) so downstream tests can
  assert the round trip without needing a real thinking-capable
  model.

### Test plan

Stub-backend round trip:

```erlang
thinking_signature_emitted_before_done_test() ->
    {ok, M} = test_load_stub_model([thinking_capable]),
    Self = self(),
    {ok, Ref} = erllama:infer(M, [1,2,3],
        #{thinking => enabled, response_tokens => 8}, Self),
    receive_until_done(Ref, []).

receive_until_done(Ref, Acc) ->
    receive
        {erllama_token, Ref, {thinking_delta, _}} = M ->
            receive_until_done(Ref, [M | Acc]);
        {erllama_thinking_end, Ref, Sig} when is_binary(Sig) ->
            ?assertNotEqual(<<>>, Sig),
            receive_until_done(Ref, [{sig, Sig} | Acc]);
        {erllama_done, Ref, _Stats} ->
            ?assert(lists:any(fun({sig, _}) -> true; (_) -> false end,
                              Acc))
    after 5000 ->
        ?assert(false)
    end.
```

## Constraints

- The downstream consumes `Stats` as an opaque map and reads keys by
  name; additive keys on `{erllama_done, _, Stats}` are
  backward-compatible.
- New message tags (`erllama_thinking_end`) must be tagged unambiguously
  so the downstream's catch-all `info(_, ...)` clauses keep working
  for older releases.
- No change to the `complete/2,3` synchronous API surface unless the
  same data is also exposed there for parity (recommended).
- Keep the dirty-CPU scheduler boundary intact; signature computation
  for the stub backend can be Erlang-native (`crypto:hash/2`).

## Verification

```bash
rebar3 fmt --check
rebar3 compile      # warnings_as_errors must stay clean
rebar3 eunit
rebar3 proper
rebar3 ct
rebar3 lint
rebar3 dialyzer
rebar3 xref

LLAMA_TEST_MODEL=/path/to/tinyllama.gguf rebar3 ct \
  --suite=test/erllama_real_model_SUITE
```

End-to-end with the downstream after bumping to the resulting
erllama release:

```python
import anthropic
c = anthropic.Anthropic(base_url="http://localhost:<port>",
                        api_key="anything")
r = c.messages.create(
    model="<alias>", max_tokens=64,
    stop_sequences=["END"],
    messages=[{"role":"user",
               "content":"reply with the literal text END and nothing else"}],
)
assert r.stop_reason == "stop_sequence"
assert r.stop_sequence == "END"
```

## Out of scope

- Vision / image / document content. Separate prompt.
- Adaptive thinking budget (`thinking.budget_tokens`). The downstream
  reads it but ignores it; once erllama grows a thinking sampler the
  caller-side budget should clip generation, but that's a separate
  ask.
- Cache token accounting (`cache_creation_input_tokens`,
  `cache_read_input_tokens` per-cache-block breakdown). The
  downstream currently emits coarse whole-prompt approximations; a
  per-call cache delta from erllama would unblock accurate
  reporting, but that's a separate prompt once the cache layer
  surfaces it.
