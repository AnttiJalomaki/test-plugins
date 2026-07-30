# Interactions API requests, responses, and streams

Source batch: `interactions-2026-05`.

Use this reference when migrating an Interactions client, reconstructing
stateless history, consuming SSE, continuing a client-side function call, or
configuring multimodal output.

## Dated migration to `steps`

Python and JavaScript SDK 2.0.0 and later select only the new schema from May 7,
2026. SDK 1.x keeps returning the legacy schema until its removal on June 8,
when Interactions calls from those versions break.

REST clients can opt in with `Api-Revision: 2026-05-20` before the May 26 default
flip. They can temporarily opt out with `Api-Revision: 2026-05-07` until June 8;
after that the legacy schema is removed and the header is ignored.

## Typed steps and stateless history

Responses use typed `steps` instead of `outputs`. Model content is nested in a
`model_output` step, while thoughts, function calls, and server-side tool
calls/results are top-level steps of their own.

```json
{
  "steps": [{
    "type": "model_output",
    "content": [{"type": "text", "text": "Hello"}]
  }]
}
```

`POST /interactions` returns output steps only. `GET /interactions/{id}` returns
the full timeline, including `user_input`. SDK callers can use `output_text`,
`output_image`, and `output_audio` for common final outputs.

For stateless history, pass the complete previous `steps` array as the next
request's `input`, then append the new turn as a `user_input` step. Function
calls remain top-level `function_call` steps. Server-executed tools create paired
typed steps such as `google_search_call` and `google_search_result`.

## Step-based SSE lifecycle

The revised lifecycle is:

```text
interaction.created
step.start -> step.delta -> step.stop
...
interaction.completed
[DONE]
```

This replaces `interaction.start`, `content.*`, and `interaction.complete`.
`interaction.completed` carries final status and usage but no `steps`, so a
streaming client must assemble steps from indexed events. Unary responses return
the assembled steps directly.

`step.delta` is discriminated by `delta.type`:

- text and base64 image data arrive as text and image deltas;
- function arguments arrive as partial argument JSON strings;
- thinking may contain `thought_summary`, followed by a final
  `thought_signature`.

Event and delta variants are extensible. Log and skip unknown types rather than
failing the stream.

## Continue a streamed function call

The call's `step.start` supplies its `id` and `name` with empty arguments.
Concatenate every matching arguments fragment and parse JSON only after the step
finishes. When the interaction completes with `requires_action`, execute the
function and begin a second streamed interaction with the first interaction ID
and matching call ID:

```json
{
  "previous_interaction_id": "v1_...",
  "input": [{
    "type": "function_result",
    "name": "get_weather",
    "call_id": "call_123",
    "result": {
      "content": [{"type": "text", "text": "{\"weather\":\"sunny\"}"}]
    }
  }],
  "stream": true
}
```

Server-side tools execute without this client round trip and expose their
call/result activity as typed steps. A request can mix them with client-side
functions, but only a client-side function leaves the interaction in
`requires_action`.

## Polymorphic `response_format`

Output controls are top-level and type-discriminated. For structured JSON,
remove `response_mime_type` and place both `mime_type` and the prior schema in a
text format:

```python
response_format={
    "type": "text",
    "mime_type": "application/json",
    "schema": {
        "type": "object",
        "properties": {"summary": {"type": "string"}},
    },
}
```

Move image settings out of `generation_config.image_config` into an image format
entry. Replace `response_modalities=["audio"]` with `{"type": "audio"}`. To
request multiple modalities, supply an array; deltas for the formats can be
interleaved in the same stream.

```python
response_format=[
    {"type": "text"},
    {
        "type": "image",
        "mime_type": "image/jpeg",
        "aspect_ratio": "1:1",
        "image_size": "1K",
    },
]
```

## Background agent streams

Agent interactions use `agent` plus `background=True` rather than `model`.
Adding `stream=True` exposes progress, thought summaries, and output through the
same step events.

```python
client.interactions.create(
    agent="deep-research-preview-04-2026",
    input="Research the topic.",
    background=True,
    stream=True,
    agent_config={"type": "deep-research", "thinking_summaries": "auto"},
)
```
