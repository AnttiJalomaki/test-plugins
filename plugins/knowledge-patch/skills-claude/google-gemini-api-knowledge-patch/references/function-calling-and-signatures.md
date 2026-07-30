# Function Calling and Thought Signatures

## Preserve signatures in manual history

Thought signatures are opaque encrypted reasoning state
(`gemini-3-thought-signatures`). Official SDKs preserve them when the complete
response object is appended to history. REST clients and manually constructed
histories must return each value unchanged on the exact model part that
received it:

```json
{
  "role": "model",
  "parts": [{
    "functionCall": {
      "name": "check_flight",
      "args": {"flight": "AA100"}
    },
    "thoughtSignature": "<opaque signature>"
  }]
}
```

For 3.x function calling, signature replay is mandatory even with minimal
thinking; omission produces HTTP 400.

Validation scans backward to the newest user message containing ordinary
content. A user message containing only a `functionResponse` does not begin a
new turn. Every subsequent step must keep the signature on its first function
call, so sequential loops resend all earlier signed model-call parts:

```text
user(text) → model(FC1 + signature A) → user(FR1)
           → model(FC2 + signature B) → user(FR2)
```

For parallel calls in one response, only the first `functionCall` part carries
the signature. Keep it on that part, return all model calls together, and then
return all function responses together:

```text
model: [FC1 + signature, FC2]
user:  [FR1, FR2]
```

Interleaving `FC1, FR1, FC2, FR2` fails validation.

## Preserve signatures in compatible chat messages and streams

OpenAI-compatible chat-completion responses attach the signature to the signed
tool call under `extra_content.google.thought_signature`. Replay that extension
with the assistant tool-call message:

```json
{
  "tool_calls": [{
    "extra_content": {
      "google": {"thought_signature": "<opaque signature>"}
    },
    "function": {
      "name": "check_flight",
      "arguments": "{\"flight\":\"AA100\"}"
    }
  }]
}
```

Without a function call, a 3.x response can put a signature on its final
content part. Replaying it is recommended for reasoning continuity but is not
validated. A streamed non-call signature can arrive on an empty-text part;
consume the stream through `finish_reason`.

With function calls, 2.5 can place an optional signature on the first part
regardless of part type. In contrast, 3.x always signs the first function-call
part and requires it on replay. Without a function call, 2.5 returns no
signature, while 3.x can sign the last part when it generated a thought.

Do not manufacture signatures for ordinary API-generated history. If an
external trace contains function-call blocks that were not produced by the API
and therefore cannot contain valid signatures, either documented sentinel can
bypass validation:

```json
{"thoughtSignature": "context_engineering_is_the_way_to_go"}
```

```json
{"thoughtSignature": "skip_thought_signature_validator"}
```

Importing unsigned function-call traces remains discouraged.

## Declare Interactions functions directly

In Interactions, a custom function is a direct typed member of `tools`
(`function-calling`), not a wrapper containing a declarations list:

```python
weather_tool = {
    "type": "function",
    "name": "get_weather",
    "description": "Get weather for a city.",
    "parameters": {
        "type": "object",
        "properties": {"city": {"type": "string"}},
        "required": ["city"],
    },
}
interaction = client.interactions.create(
    model="MODEL_ID",
    input="Weather in Paris?",
    tools=[weather_tool],
)
```

Set behavior with `generation_config.tool_choice`:

- `auto` is the default.
- `any` forces a function call.
- `none` prohibits calls.
- Preview mode `validated` enforces schema adherence.

Restrict callable functions with nested `allowed_tools`:

```python
generation_config = {
    "tool_choice": {
        "allowed_tools": {"mode": "any", "tools": ["get_weather"]}
    }
}
```

Mode `any` can reject very large or deeply nested schemas.

## Return typed and multimodal function results

For 3-series models, a `function_result` can carry multiple typed blocks,
including images. Preserve the function name and call ID:

```python
input=[{
    "type": "function_result",
    "name": tool_call.name,
    "call_id": tool_call.id,
    "result": [
        {"type": "text", "text": "instrument.jpg"},
        {
            "type": "image",
            "mime_type": "image/jpeg",
            "data": base64_data,
        },
    ],
}]
```

When using legacy `generateContent` with Gemini 3.x, every `FunctionResponse`
also requires both `call_id` and function `name` (`gemini-3.6`).

## Connect remote MCP tools

Interactions accepts an `mcp_server` tool:

```python
tools=[{
    "type": "mcp_server",
    "name": "weather_service",
    "url": "https://example.com/mcp",
    "allowed_tools": ["get_weather"],
}]
```

Only Streamable HTTP is supported; SSE transport is not. Server names cannot
contain hyphens. Use optional `headers` for authentication and `allowed_tools`
for filtering.

## Assemble streamed argument deltas

SDK-normalized streamed Interactions events provide the call ID and name at
`step.start`. Group calls by `event.index`, append
`event.delta.partial_arguments` when `event.delta.type == "arguments"`, and
parse only after `interaction.completed`:

```python
if (
    event.event_type == "step.delta"
    and event.delta.type == "arguments"
):
    current_calls[event.index]["arguments"] += (
        event.delta.partial_arguments
    )
```

The revisioned SSE wire form calls this discriminant `arguments_delta` and
places the fragment in `arguments`; see the Interactions reference. Do not mix
field names between raw-event and SDK-object consumers.

## Avoid structured text immediately before a tool call

Requiring XML, YAML, or JSON text immediately before calling a tool can produce
`Malformed_Function_Call`. Prefer a dedicated function for working notes,
called alongside the real function:

```json
{
  "name": "update",
  "description": "Record working notes before another tool call.",
  "parameters": {
    "type": "OBJECT",
    "properties": {
      "next_step": {"type": "STRING"}
    },
    "required": ["next_step"]
  }
}
```

Markdown notes or removing the pre-tool text requirement are fallback options.

## Combine built-in and custom tools

A single request can include built-in tools and custom functions. Computer Use
is in public preview on `gemini-3.5-flash`, with browser, mobile, and desktop
environments plus configurable safety and prompt-injection controls
(`release-lifecycle`).
