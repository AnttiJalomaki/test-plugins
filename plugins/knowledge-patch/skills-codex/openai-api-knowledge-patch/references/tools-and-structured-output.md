# Tools and Structured Output

## Responses structured output

Place raw structured-output formats under `text.format`, including JSON mode:

```json
{
  "text": {
    "format": {"type": "json_object"}
  }
}
```

For SDK parsing, pass a Pydantic model through Python `text_format`, or a Zod
format through JavaScript `text.format`.

```python
response = client.responses.parse(
    model="gpt-5.6",
    input=[{"role": "user", "content": "Extract the result."}],
    text_format=Result,
)

for output in response.output:
    if output.type == "message":
        for item in output.content:
            value = item.refusal if item.type == "refusal" else item.parsed
```

Do not assume every message content item conforms to the requested schema. A
safety refusal is emitted as a distinct `refusal` item.

## Function-call round trip

Each Responses `function_call` output item includes `name`, JSON-encoded
`arguments`, and `call_id`. Preserve all output items in the running input,
execute each call, and append its result as `function_call_output` with the
same `call_id`.

```python
input_list += response.output
for call in response.output:
    if call.type == "function_call":
        result = run(call.name, json.loads(call.arguments))
        input_list.append({
            "type": "function_call_output",
            "call_id": call.call_id,
            "output": json.dumps(result),
        })
```

The output is normally a string. It can instead be an array containing
supported image or file objects.

## Namespaces and deferred loading

Group related functions in a `namespace`. Mark a function
`defer_loading: true` to omit it from initial context and let `tool_search`
discover it. Tool search requires GPT-5.4 or later.

```json
{
  "type": "namespace",
  "name": "crm",
  "tools": [{
    "type": "function",
    "name": "lookup",
    "defer_loading": true,
    "parameters": {"type": "object", "properties": {}}
  }]
}
```

Before the eventual `function_call`, the response may contain
`tool_search_call` and `tool_search_output`. Preserve both in interaction
history.

## Per-turn allowed tools

Keep the full tool list stable and use `tool_choice` of type `allowed_tools` to
restrict the callable subset for one turn. This avoids modifying the full list
and preserves prompt-cache reuse.

```json
{
  "type": "allowed_tools",
  "mode": "auto",
  "tools": [
    {"type": "function", "name": "get_weather"},
    {"type": "function", "name": "search_docs"}
  ]
}
```

When tool search is enabled, the allowed-tools restriction applies only to the
tools currently loaded for that turn.

## Parallel-call edge cases

Built-in tools cannot be combined with parallel function calling.
`parallel_tool_calls: false` limits the turn to zero or one call.

When a fine-tuned model emits multiple calls, strict mode is disabled for
those calls. The `gpt-4.1-nano-2025-04-14` snapshot can also duplicate a tool
call when parallel calls are enabled; make execution idempotent or deduplicate
by call identity.

## Function-call streaming

A streamed function call follows this event sequence:

1. `response.output_item.added` starts the item.
2. `response.function_call_arguments.delta` emits JSON fragments.
3. `response.function_call_arguments.done` supplies the complete encoded
   arguments.
4. `response.output_item.done` completes the item.

Aggregate fragments by `output_index`. Associate the aggregate with its call
using `item_id`.

## Free-form custom tools

A tool with `type: "custom"` accepts arbitrary text rather than JSON-schema
arguments:

```json
{
  "type": "custom",
  "name": "code_exec",
  "description": "Executes Python code."
}
```

The emitted `custom_tool_call` item places that text in `input` and also
provides `name` and `call_id`.

## Grammar-constrained custom tools

Set a custom tool's `format` to type `grammar`, with syntax `lark` or `regex`.
Both regex-based formats use Rust regex syntax, which does not support
lookarounds or lazy modifiers.

```json
{
  "type": "custom",
  "name": "date",
  "format": {
    "type": "grammar",
    "syntax": "regex",
    "definition": "^[0-9]{4}-[0-9]{2}-[0-9]{2}$"
  }
}
```

The Lark subset omits terminal priorities, templates, `%declare`, and imports
other than common imports. Keep terminals bounded, write whitespace
explicitly, and put an anchored free-form span in one terminal because greedy
lexing happens before parsing.
