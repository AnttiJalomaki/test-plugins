# Function calling and tools

Source batch: `function-calling`.

Use this reference when declaring Interactions tools, controlling tool choice,
returning multimodal results, connecting an MCP server, or reconstructing
streamed arguments.

## Direct function declarations

Interactions custom functions are direct typed entries in the `tools` array,
not wrappers around a declarations list. `parameters` is an object schema with
`properties` and `required`.

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
    model="MODEL_ID", input="Weather in Paris?", tools=[weather_tool]
)
```

## Tool choice and validation

Set behavior through `generation_config.tool_choice`:

- `auto` is the default;
- `any` forces a call;
- `none` prohibits calls;
- preview `validated` enforces schema adherence.

Restrict functions with the nested `allowed_tools` object. `any` can reject
very large or deeply nested schemas.

```python
generation_config = {
    "tool_choice": {
        "allowed_tools": {"mode": "any", "tools": ["get_weather"]}
    }
}
```

## Multimodal function results

For Gemini 3-series models, an Interactions `function_result` can contain
multiple typed content blocks, including images. Put the blocks in `result` and
preserve both the function name and call ID when continuing.

```python
input=[{
    "type": "function_result",
    "name": tool_call.name,
    "call_id": tool_call.id,
    "result": [
        {"type": "text", "text": "instrument.jpg"},
        {"type": "image", "mime_type": "image/jpeg", "data": base64_data},
    ],
}]
```

For Gemini 3.x on `generateContent`, every `FunctionResponse` likewise needs
both its `call_id` and function `name`.

## Remote MCP tools

Interactions can connect directly to a remote MCP server with an `mcp_server`
tool. Only Streamable HTTP is supported, not SSE. The server name must not
contain hyphens. Optional `headers` and `allowed_tools` fields carry
authentication data and tool filters.

```python
tools=[{
    "type": "mcp_server",
    "name": "weather_service",
    "url": "https://example.com/mcp",
    "allowed_tools": ["get_weather"],
}]
```

## Stream argument deltas

At `step.start`, capture each function's ID and name and associate it with
`event.index`. Append each subsequent argument fragment for that index. In the
Python event representation, the fragment arrives when
`event.delta.type == "arguments"` at `event.delta.partial_arguments`:

```python
if event.event_type == "step.delta" and event.delta.type == "arguments":
    current_calls[event.index]["arguments"] += event.delta.partial_arguments
```

Serialized lifecycle descriptions may represent the same incremental data as
an `arguments_delta.arguments` fragment. In either representation, do not parse
until the step or interaction has completed and all fragments have been joined.

## Avoid structured pre-tool text

Requiring XML, YAML, or JSON text immediately before a tool call can cause
`Malformed_Function_Call`. Prefer making working notes a dedicated function
call alongside the real call. Markdown notes or dropping the pre-tool text
requirement are fallback options.

```json
{
  "name": "update",
  "description": "Record working notes before another tool call.",
  "parameters": {
    "type": "OBJECT",
    "properties": {"next_step": {"type": "STRING"}},
    "required": ["next_step"]
  }
}
```
