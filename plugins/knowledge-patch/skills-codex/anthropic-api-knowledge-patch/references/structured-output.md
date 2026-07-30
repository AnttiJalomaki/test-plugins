# Structured Output and Schema Contracts

## Raw JSON output

Structured output is GA and needs no beta header. Put a JSON Schema in
`output_config.format` with `type: "json_schema"`. The matching JSON is still
returned as a string in a text content block, so raw callers must locate that
block and decode its text.

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

The Python convenience API remains an intentional exception:
`client.messages.parse()` accepts `output_format=SomePydanticModel`, translates
it internally, and returns the validated value as `response.parsed_output`.

```python
response = client.messages.parse(
    model=model_id,
    max_tokens=256,
    messages=[{"role": "user", "content": "Extract the order number."}],
    output_format=Order,
)
order = response.parsed_output
```

Other SDKs require `output_config` directly.

## SDK schema simplification

Python, TypeScript, Ruby, and PHP helpers strip unsupported constraints such
as `minimum`, `maximum`, `minLength`, and `maxLength`, move their intent into
descriptions, add `additionalProperties: false`, and remove unsupported string
formats before sending the schema. C# and Go do the same when deriving schemas
from native types.

These SDKs validate the response against the original schema client-side.
The server grammar, however, is constrained by the simplified schema, not
every constraint from the source type.

## Strict tool input

Set `strict: true` on each tool to grammar-constrain the selected tool name and
input to its `input_schema`. Strict and non-strict tools may coexist, and
strict tools may be combined with a final JSON output schema.

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

The final-output grammar applies only to direct output. It does not constrain
tool calls, tool results, or thinking, so every constrained tool needs its own
`strict: true`.

## Combined complexity ceilings

The following limits are counted across all strict tool schemas and the final
output schema together:

- At most 20 strict tools.
- At most 24 optional parameters.
- At most 16 parameters that use `anyOf` or a type array.

Other combinations of unions, nesting, optionals, and tool count can still
return HTTP 400 `Schema is too complex for compilation`. Grammar compilation
times out after 180 seconds.

## Parsing and compliance edge cases

Inspect `stop_reason` before decoding. A refusal can return content outside the
schema, and a `max_tokens` stop can leave it incomplete.

Even after a normal completion, an `enum` or `const` value can differ from its
declaration in capitalization. Avoid schema values distinguished only by
case, and compare returned values case-insensitively.

Object output ordering is deterministic: required properties appear first in
schema order, followed by optional properties in schema order.

## Incompatible features

- Combining citations with `output_config.format` returns HTTP 400.
- Assistant-message prefilling is incompatible with JSON output.
- A final-output schema does not make tool inputs strict.

## Sensitive schema data

Message prompts and responses may retain zero-data-retention handling, but the
schema-derived grammar is cached separately for up to 24 hours and does not
receive the same protected-health-information safeguards.

Keep PHI out of property names, `enum` values, `const` values, and regex
patterns. Put sensitive values only in message content.
