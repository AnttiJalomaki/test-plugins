# Responses, State, and Safety

## Request shape and candidate generation

Responses produces exactly one generation per request and does not accept the
Chat Completions `n` parameter. Send independent requests when the application
needs multiple candidate outputs. (`responses-api`; 2025-03-11)

## Response chains and instructions

Use `previous_response_id` to carry prior response context. It does not carry
top-level `instructions`, so resend stable instructions on every request.
Tokens from earlier input in the chained context are still billed as input
tokens.

## Storage and stateless reasoning

Responses are stored by default. Chat Completions are also stored by default
for new accounts. Set `store: false` when the application must remain
stateless. Zero Data Retention flows enforce disabled storage automatically.

When continuing reasoning without storage, replay every reasoning item returned
by the preceding response, including its default `encrypted_content`. Do not
reduce the replay to assistant message text.

## Function strictness

Responses internally tags function definitions. If `strict` is omitted, the
API attempts strict mode rather than preserving the old non-strict default.
When a schema is incompatible with strict mode, calling falls back to best
effort and the returned definition reports `strict: false`.

Set non-strict behavior explicitly:

```json
{
  "type": "function",
  "name": "lookup",
  "parameters": {
    "type": "object",
    "properties": {}
  },
  "strict": false
}
```

## Chained reasoning context

For GPT-5.6, set `reasoning.context` according to the validity of prior goals
and assumptions:

- Use `all_turns` with `previous_response_id` while the reasoning context
  remains applicable.
- Use `current_turn` when earlier reasoning has become stale.
- Use `auto`, or omit the property, to accept the model default; inspect the
  returned effective value.

```json
{
  "reasoning": {"context": "all_turns"},
  "previous_response_id": "resp_..."
}
```

When manually replaying the history, retain every user input and every output
item. Preserve item IDs, call IDs, caller metadata, and assistant phase values;
these are part of the continuation record.

## Streaming transports

An HTTP request with `stream=true` streams server-sent events. Persistent
WebSocket mode supports incremental inputs and chains them with
`previous_response_id`.

If moderation scores are requested with a generation, they arrive only after
the full output. Partial deltas do not include those scores, so do not present
the streamed content as having passed the later moderation result.

## Programmatic tool calling

Use the hosted program tool for bounded reductions over tool results. Enable
the tool and opt individual functions into programmatic invocation with
`allowed_callers`:

```json
{
  "tools": [
    {"type": "programmatic_tool_calling"},
    {
      "type": "function",
      "name": "lookup_records",
      "allowed_callers": ["programmatic"]
    }
  ]
}
```

Process and preserve all `program`, program-issued `function_call`,
`function_call_output`, and `program_output` items. Keep `call_id` and `caller`
unchanged through the host round trip.

## Multi-agent Responses beta

Enable the beta using both the request header and a bounded concurrency value:

```text
OpenAI-Beta: responses_multi_agent=v1
```

```json
{
  "multi_agent": {
    "enabled": true,
    "max_concurrent_subagents": 3
  }
}
```

Handle and replay `multi_agent_call`, `multi_agent_call_output`, and
`agent_message` items. Also handle function calls issued by any subagent, not
only calls emitted by the parent.

## Generation-time safeguards

Cyber and biology safeguards can synchronously review generation. A request,
including a legitimate dual-use request, can be refused or its stream can pause
for several seconds during review. Make clients tolerant of the pause and
refusal path.

Attach a stable, privacy-preserving `safety_identifier` for each end user to
every Responses request. Realtime uses a different transport described in
[service-tiers-and-realtime.md](service-tiers-and-realtime.md).
