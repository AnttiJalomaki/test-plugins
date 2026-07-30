# Interactions API

## Migrate to the steps schema

The dated Interactions migration (`interactions-2026-05`) has three client
paths:

- Python and JavaScript SDK 2.0.0 and later select the new schema from May 7,
  2026.
- SDK 1.x continues returning the legacy schema until June 8, 2026; after the
  legacy schema is removed, Interactions calls from those versions break.
- REST can opt in early with `Api-Revision: 2026-05-20` before the May 26
  default flip. It can temporarily opt out with `Api-Revision: 2026-05-07`
  until June 8; after that, the legacy schema is unavailable and the header is
  ignored.

Responses now expose typed `steps` instead of flat `outputs`:

```json
{
  "steps": [{
    "type": "model_output",
    "content": [{"type": "text", "text": "Hello"}]
  }]
}
```

Model content is nested in `model_output`. Thoughts, function calls, and
server-side tool calls and results are top-level typed steps. `POST
/interactions` returns output steps only; `GET /interactions/{id}` returns the
complete timeline, including `user_input`. SDK convenience accessors
`output_text`, `output_image`, and `output_audio` expose common final outputs.

For stateless history, send the complete preceding `steps` array as the next
request's `input`, then append the new turn as a `user_input` step. A function
call remains a top-level `function_call`; server tools produce paired steps
such as `google_search_call` and `google_search_result`.

## Assemble a streamed interaction

The step-based SSE lifecycle is:

```text
interaction.created
step.start
step.delta
step.stop
...
interaction.completed
[DONE]
```

This replaces `interaction.start`, `content.*`, and `interaction.complete`.
The completion event contains final status and usage but no `steps`; a client
must assemble steps from indexed events. A unary response already has the
assembled `steps`.

Dispatch `step.delta` by its discriminant. In the revisioned wire shape:

- `text` carries text.
- `image` carries base64 image data.
- `arguments_delta` carries a partial JSON string in `arguments`.
- `thought_summary` is optional thinking output and the final thinking event
  can include `thought_signature`.

Event and delta variants are extensible. Log and skip unknown types rather than
terminating the stream.

For a function call, `step.start` provides its `id` and `name` with empty
arguments. Concatenate every matching argument fragment and parse the JSON only
after the step finishes.

## Continue after `requires_action`

If the first interaction completes with `requires_action`, run the requested
client function and create a second streamed interaction:

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

Match both the interaction ID and call ID. Server-side tools execute without a
client round trip and expose their work as typed call/result steps. A request
can combine server-side tools and client functions, but only client functions
leave it in `requires_action`.

## Configure typed output modalities

Output settings moved to a top-level, type-discriminated `response_format`.
Structured JSON no longer uses a top-level `response_mime_type`; put
`mime_type` and `schema` under a text format:

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

Image options moved out of `generation_config.image_config` into an image
format entry. Audio replaces `response_modalities=["audio"]` with
`{"type": "audio"}`. Request multiple outputs with an array:

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

Different modalities can produce interleaved deltas in one stream; route each
delta by type and index.

## Run managed agents in the background

Agent interactions use `agent` rather than `model` and require
`background=True`. Add `stream=True` to receive progress, thought summaries,
and output through the same step lifecycle:

```python
client.interactions.create(
    agent="deep-research-preview-04-2026",
    input="Research the topic.",
    background=True,
    stream=True,
    agent_config={"type": "deep-research", "thinking_summaries": "auto"},
)
```

The `antigravity-preview-05-2026` managed agent uses 3.6 Flash by default and
can run in the remote environment:

```python
client.interactions.create(
    agent="antigravity-preview-05-2026",
    input="Complete the browser task.",
    environment="remote",
)
```

New Deep Research variants support collaborative planning, visualization, MCP
server integration, File Search, streamed UI results, and a more comprehensive
automated research path.

## Account for operational field changes

In v1beta, `total_reasoning_tokens` was renamed to `total_thought_tokens`.
Supported Interactions calls also expose developer logs for operational
inspection (`release-lifecycle`).
