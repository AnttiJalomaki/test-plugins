# Tools and Structured Output

Use this reference for Responses function calls, structured parsing, tool discovery, constrained tool sets, streaming, custom tools, Programmatic Tool Calling, and the multi-agent beta.

## Function definition strictness

Responses internally tags function definitions. When `strict` is omitted, it attempts strict mode rather than preserving the older non-strict default. If a schema is incompatible with strict mode, the API falls back to best-effort calling and reports `strict: false`.

Set non-strict behavior explicitly when it is intentional:

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

Do not infer that an omitted `strict` guarantees strict execution; inspect the effective result when schema compatibility is uncertain.

## Structured response parsing

Raw Responses structured-output formats belong below `text.format`. JSON mode uses:

```json
{
  "text": {
    "format": {"type": "json_object"}
  }
}
```

SDK parse helpers use different convenience fields:

- Python passes a Pydantic model as `text_format`.
- JavaScript passes a Zod format under `text.format`.

A safety refusal is a distinct `refusal` content Item, not data matching the requested schema. Inspect message content before consuming parsed output.

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

## Function-call round trip

A `function_call` output Item contains `name`, JSON-encoded `arguments`, and `call_id`. Preserve all response output Items in the next input, execute each applicable call, and append a `function_call_output` with exactly the same `call_id`.

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

The function result is normally a string. It may instead be an array containing image or file objects, so avoid string-only validators when the tool can return those media types.

## Namespaces and deferred loading

A `namespace` groups related functions. Set `defer_loading: true` on functions that should stay out of initial context until `tool_search` discovers them. Tool search requires `gpt-5.4` or later.

```json
{
  "type": "namespace",
  "name": "crm",
  "tools": [
    {
      "type": "function",
      "name": "lookup",
      "defer_loading": true,
      "parameters": {"type": "object", "properties": {}}
    }
  ]
}
```

Before the eventual `function_call`, the output can contain `tool_search_call` and `tool_search_output`. Retain both Items in interaction history.

## Per-turn allowed tools

Keep a stable full tool list for prompt-cache reuse and narrow the callable subset per turn with `tool_choice`:

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

With tool search, the restriction applies only to tools already loaded for that turn. A deferred tool outside the current loaded set is not made callable merely by naming it in the restriction.

## Parallel-call edge cases

- Built-in tools cannot be combined with parallel function calling.
- `parallel_tool_calls: false` limits a turn to zero or one call.
- Multiple calls from a fine-tuned model disable strict mode for those calls.
- `gpt-4.1-nano-2025-04-14` can duplicate a tool call when parallel calls are enabled.

Tool executors should deduplicate by protocol identity or make side effects idempotent rather than assuming each emitted call is unique.

## Function-call streaming

A streamed function call follows this event sequence:

1. `response.output_item.added` announces the Item.
2. `response.function_call_arguments.delta` sends fragments of JSON-encoded arguments.
3. `response.function_call_arguments.done` supplies the complete encoded arguments.
4. `response.output_item.done` completes the Item.

Aggregate deltas by `output_index`. Use `item_id` to associate fragments and completion with the correct call. Execute from the complete encoded arguments rather than assuming each delta is independently valid JSON.

## Free-form custom tools

A `custom` tool receives arbitrary text rather than JSON-schema arguments:

```json
{
  "type": "custom",
  "name": "code_exec",
  "description": "Executes Python code."
}
```

The corresponding output Item is `custom_tool_call` and includes `name`, `call_id`, and the text in `input`.

## Grammar-constrained custom tools

Set a custom tool's `format` to `grammar` and choose `lark` or `regex`:

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

Both formats use Rust regex syntax, which excludes lookarounds and lazy modifiers. The Lark implementation also excludes terminal priorities, templates, `%declare`, and non-common imports. Keep terminals bounded, make whitespace explicit, and place an anchored free-form span in one terminal because greedy lexing happens before parsing.

## Programmatic Tool Calling

Use Programmatic Tool Calling for bounded reductions over function results. Enable the hosted program tool and opt individual functions in through `allowed_callers`:

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

The host must process and preserve `program`, program-issued `function_call`, `function_call_output`, and `program_output` Items. Preserve both `call_id` and `caller` so calls and outputs remain associated with the program.

## Multi-agent Responses beta

Enable the beta with both the request header and bounded concurrency:

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

Handle and replay `multi_agent_call`, `multi_agent_call_output`, and `agent_message` Items. Also handle function calls issued by any subagent; do not assume only the top-level agent can call a tool.

## Host checklist

1. Validate requested strictness and inspect fallback behavior.
2. Branch parsed content between `refusal` and schema-shaped data.
3. Preserve every output Item and correlate calls by `call_id`.
4. Retain tool-search, program, and multi-agent protocol Items.
5. Restrict tools without rebuilding the stable full list.
6. Make parallel tool side effects idempotent.
7. Aggregate streamed arguments by `output_index` and `item_id`.
