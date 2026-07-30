# Structured Outputs

Use this reference for schema-constrained final responses and strict tool
arguments.

## Raw JSON output

Structured output is GA and needs no beta header. Set
`output_config.format.type` to `json_schema` and provide `schema`. The response
still carries matching JSON as a string in a text content block; a raw client
must select that block and decode its text.

```python
response = client.messages.create(
    model=model_id,
    max_tokens=256,
    messages=[{"role": "user", "content": "Extract the order number."}],
    output_config={"format": {
        "type": "json_schema",
        "schema": {
            "type": "object",
            "properties": {"order_number": {"type": "string"}},
            "required": ["order_number"],
            "additionalProperties": False,
        },
    }},
)
data = json.loads(next(b.text for b in response.content if b.type == "text"))
```

The Python convenience parser is an intentional exception. Pass a Pydantic
model through `output_format` to `client.messages.parse()`; it translates the
request and returns the validated object as `response.parsed_output`. Other
SDKs require `output_config` directly.

```python
response = client.messages.parse(
    model=model_id,
    max_tokens=256,
    messages=[{"role": "user", "content": "Extract the order number."}],
    output_format=Order,
)
order = response.parsed_output
```

## SDK schema simplification

Python, TypeScript, Ruby, and PHP helpers strip unsupported constraints such as
`minimum`, `maximum`, `minLength`, and `maxLength`, move their intent into
descriptions, add `additionalProperties: false`, and filter unsupported string
formats. C# and Go do the same when deriving schemas from native types.

These SDKs validate the result against the original schema locally, but the
server grammar sees only the simplified schema. Do not assume every source-type
constraint influenced generation.

## Strict tools

Set `strict: true` on a tool to grammar-constrain its selected name and input to
`input_schema`. Strict and non-strict tools may coexist, and strict tools can be
combined with a final JSON output schema.

```python
tools=[{
    "name": "lookup_order",
    "strict": True,
    "input_schema": {
        "type": "object",
        "properties": {"order_number": {"type": "string"}},
        "required": ["order_number"],
        "additionalProperties": False,
    },
}]
```

The final-output grammar constrains only direct output. It does not constrain
tool calls, tool results, or thinking; tool arguments need their own
`strict: true` schema.

## Complexity ceilings

Across all strict tool schemas plus the final-output schema, one request may
contain at most:

- 20 strict tools
- 24 optional parameters
- 16 parameters using `anyOf` or a type array

Union interactions, nesting, optional fields, and tool count can still cause
HTTP 400 `Schema is too complex for compilation`. Grammar compilation times out
after 180 seconds.

## Parsing and compliance edge cases

Inspect `stop_reason` before parsing. Refusals can return non-schema content,
and `max_tokens` can truncate a response before it satisfies the schema.

Even a normally completed `enum` or `const` may differ from the declared value
in capitalization. Avoid values distinguished only by case and compare these
values case-insensitively. Objects emit required properties first in schema
order, followed by optional properties in schema order.

Citations combined with `output_config.format` return HTTP 400. Assistant
message prefilling is also incompatible with JSON output.

## Sensitive data in schema grammars

Prompts and responses can retain zero-data-retention handling, but the compiled
schema grammar is cached separately for up to 24 hours and does not receive the
same protected-health-information safeguards. Keep PHI out of property names,
`enum` and `const` values, and regex patterns; place it only in message content.
