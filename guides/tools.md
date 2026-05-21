# Server-side tools

Most tool calls are *client-executed*: the model emits a tool call,
the server returns it, and your client runs the tool and sends the
result back on the next request. erllama_server can also run a
**built-in tool server-side**: when the model calls it, the server
executes the tool in-process, feeds the result back into the
conversation, and continues generating, until the model answers
without calling a server tool (or a per-request cap is reached). This
works on `/v1/responses`, `/v1/chat/completions`, and `/v1/messages`.

Nothing runs server-side unless you register an executor. The shipped
example is `web_search`.

## Enable web_search

Register an executor in `config/sys.config` under
`builtin_tool_executors`, keyed by the tool `type`. Pick one provider:

```erlang
{erllama_server, [
  %% ... other settings ...
  {builtin_tool_executors, #{
    <<"web_search">> => #{
      module   => erllama_server_tool_executor_web_search,
      type     => <<"web_search">>,
      provider => tavily,            %% tavily | ollama | brave | searxng
      api_key  => <<"tvly-...">>
    }
  }},
  %% Cap on server-side tool rounds per request (default 5).
  {max_tool_iterations, 5}
]}
```

Provider config:

| provider  | keys                                  | backend                          |
|-----------|---------------------------------------|----------------------------------|
| `tavily`  | `api_key`                             | `api.tavily.com` (LLM-tuned)     |
| `ollama`  | `api_key`                             | `ollama.com/api/web_search`      |
| `brave`   | `api_key`                             | Brave Search API                 |
| `searxng` | `endpoint` (e.g. `http://host:8888`)  | your self-hosted SearXNG         |

Optional keys for any provider: `max_results` (default 5),
`timeout_ms` (default 10000), `endpoint` (override the default URL).

Restart the server after editing `sys.config`.

## Test it

With an executor registered, a model that decides to search will have
the search run server-side and the answer come back already grounded
in results. Force a search to verify the path end-to-end by sending
the tool plus a tool-choice that requires it:

```sh
# OpenAI Responses surface
curl -N -sX POST http://127.0.0.1:8080/v1/responses \
  -H 'content-type: application/json' \
  -d '{
    "model":"gpt-4o",
    "input":"What is the capital of France?",
    "tools":[{"type":"web_search"}],
    "tool_choice":"required",
    "max_output_tokens":128,
    "stream":true
  }'
```

```sh
# OpenAI chat surface
curl -sX POST http://127.0.0.1:8080/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{
    "model":"gpt-4o",
    "messages":[{"role":"user","content":"Latest news about Erlang/OTP?"}],
    "tools":[{"type":"web_search"}],
    "tool_choice":"required",
    "max_tokens":128
  }'
```

```sh
# Anthropic surface (versioned built-in type is normalised to web_search)
curl -sX POST http://127.0.0.1:8080/v1/messages \
  -H 'content-type: application/json' \
  -d '{
    "model":"claude-sonnet-4-5","max_tokens":256,
    "messages":[{"role":"user","content":"Who won the 2026 ... ?"}],
    "tools":[{"type":"web_search_20250305","name":"web_search"}],
    "tool_choice":{"type":"any"}
  }'
```

What to expect:

- The server runs the search in-process (no tool call is returned to
  you); the response text reflects the results.
- The search tool round is invisible on the wire (chat/messages) or
  surfaces as a `web_search_call` item (responses); the client just
  sees the final answer.
- A model that keeps calling the tool is bounded by
  `max_tool_iterations`; the turn then finishes with a
  length/`max_tokens` finish reason.

Provider-side checks: an unregistered/disabled `web_search` is simply
dropped from the request (no error, no search). A misconfigured
provider (missing `api_key`, unreachable `endpoint`) folds an error
result into the conversation so the model can recover rather than
failing the turn; the server logs the failure.

## Add your own executor

A new server-side tool is one module plus one registry line. The
module implements the `erllama_server_tool_executor` behaviour:

```erlang
-module(my_tool_executor).
-behaviour(erllama_server_tool_executor).
-export([declare/0, execute/2]).

%% Model-facing name + JSON-Schema for the arguments.
declare() ->
    #{name => <<"my_tool">>,
      description => <<"...">>,
      schema => #{<<"type">> => <<"object">>,
                  <<"properties">> => #{<<"x">> => #{<<"type">> => <<"string">>}},
                  <<"required">> => [<<"x">>]}}.

%% Run it. Args is the parsed arguments map; Ctx carries model,
%% request_id, session_id, and config (the registry entry's extra
%% keys). Return a JSON-encodable map, or {error, Reason}.
execute(#{<<"x">> := X}, _Ctx) ->
    {ok, #{<<"result">> => X}}.
```

Then register it: `<<"my_tool">> => #{module => my_tool_executor,
type => <<"my_tool">>}`. The grammar offers it, the loop runs it, and
it works on every API surface.
