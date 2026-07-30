# Structured outputs

Source batch: `structured-outputs`.

Use this reference when defining a recursive schema, validating streamed JSON,
combining a built-in tool with a constrained final response, or checking the
supported schema subset.

## Recursive response schemas

Structured-output schemas can recurse. To recurse to the root, use `$ref` from
the nested value back to `#`. A self-referential Pydantic model can provide the
equivalent schema through `model_json_schema()`.

```json
{
  "type": "object",
  "properties": {
    "name": {"type": "string"},
    "children": {"type": "array", "items": {"$ref": "#"}}
  },
  "required": ["name", "children"]
}
```

## Accumulate stream fragments

With `stream=True`, structured output arrives in text step deltas as partial
JSON strings. Concatenate them in order and validate only the completed
document.

```python
fragments = []
for event in stream:
    if event.event_type == "step.delta" and event.delta.type == "text":
        fragments.append(event.delta.text)
result = Result.model_validate_json("".join(fragments))
```

Do not parse each fragment as standalone JSON and do not validate until the
stream has supplied the full document.

## Built-in tools plus a structured final response

As a preview limited to Gemini 3-series models, an Interactions request can run
built-in tools and still constrain its final text to a response schema. Supply
`tools` and the JSON `response_format` together.

```python
interaction = client.interactions.create(
    model="MODEL_ID",
    input="Read the supplied URL and extract the result.",
    tools=[{"type": "url_context"}],
    response_format={
        "type": "text",
        "mime_type": "application/json",
        "schema": Result.model_json_schema(),
    },
)
```

## Supported subset and complexity limits

In addition to basic types and constraints, the supported subset includes:

- nullable unions such as `{"type": ["string", "null"]}`;
- schema-valued or boolean-valued `additionalProperties`;
- string formats `date-time`, `date`, and `time`;
- numeric `minimum` and `maximum`;
- array `prefixItems`, `minItems`, and `maxItems`.

Very large or deeply nested schemas can be rejected. Keep schemas as compact as
the response contract permits.
