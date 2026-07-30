# Structured Outputs

## Define recursive schemas

Structured-output schemas can recurse (`structured-outputs`). To recurse to the
root, use `$ref: "#"`:

```json
{
  "type": "object",
  "properties": {
    "name": {"type": "string"},
    "children": {
      "type": "array",
      "items": {"$ref": "#"}
    }
  },
  "required": ["name", "children"]
}
```

A self-referential Pydantic model can supply the equivalent schema through
`model_json_schema()`.

## Accumulate streamed JSON before validation

With `stream=True`, structured output arrives as partial JSON strings in text
step deltas. Preserve order, concatenate every fragment, and validate only the
completed document:

```python
fragments = []
for event in stream:
    if (
        event.event_type == "step.delta"
        and event.delta.type == "text"
    ):
        fragments.append(event.delta.text)

result = Result.model_validate_json("".join(fragments))
```

Do not parse individual deltas as standalone JSON.

## Combine built-in tools with a structured final response

As a preview limited to 3-series models, an interaction can run built-in tools
and constrain its final text to a response schema. Supply `tools` and the JSON
`response_format` in the same request:

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

## Stay within the supported schema subset

Along with basic JSON types and constraints, supported features include:

- nullable unions such as `{"type": ["string", "null"]}`
- schema-valued or boolean-valued `additionalProperties`
- string formats `date-time`, `date`, and `time`
- numeric `minimum` and `maximum`
- array `prefixItems`, `minItems`, and `maxItems`

Very large or deeply nested schemas can be rejected. Keep schemas as small as
the output contract permits.
