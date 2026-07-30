# Rego language and built-ins

## Rego v1 syntax and compiler rules

Since `1.0-migration`, `in`, `every`, `if`, and `contains` are keywords by
default; the corresponding `future.keywords` imports are no-ops. A rule with a
body needs `if`, a multi-value rule needs `contains`, value assignment remains
valid without `if`, and a solitary reference head such as `p.a` is invalid.

```rego
package example

allow if {
    input.user == "alice"
}

reasons contains "missing role" if {
    not input.role
}

limit := 10
```

Compiler defaults also reject duplicate or shadowing imports and declarations
that use `input` or `data` as rule or variable names. `with input as ...` and
`with data as ...` still replace those documents during evaluation.

The deprecated built-ins removed for v1 are `any`, `all`, `re_match`,
`net.cidr_overlap`, `set_diff`, `cast_array`, `cast_set`, `cast_string`,
`cast_boolean`, `cast_null`, and `cast_object`.

## Parsing and references

Since 1.6.0, keyword-named path segments such as `package`, `if`, `else`, and
`not` work in dotted references rather than requiring bracket notation:

```rego
package example

allow if {
    input.package.source == "internal"
}
```

Also since 1.6.0, primitive numbers with surplus leading zeros always fail to
parse. Replace `0123` with `123`. Excessively nested input is rejected by the
parser's recursion-depth guard since 1.5.0; callers must handle that parse error.

## String and number built-ins

- `units.parse` accepts scientific notation such as `units.parse("1e3KB")`
  since 1.0.0.
- `to_number` rejects `"Inf"`, `"Infinity"`, and `"NaN"` since 1.0.0.
- `regex.replace`, `replace`, `strings.replace_n`, and `concat` observe
  evaluation cancellation since 1.12.0.
- `strings.render_template` no longer has a fixed missing-key error as of
  1.13.0. Do not match the old error text.

OPA 1.12.0 added interpolated string templates. Prefix the quoted string with
`$` and place expressions in braces. An undefined expression renders
`<undefined>` rather than making the rule undefined.

```rego
package messages

message := $"User {input.username} has role {input.role}"
```

## Arrays, graphs, and durations

OPA 1.13.0 introduced `array.flatten`, but that release mishandles single-item
arrays. Use 1.13.1 or later.

```rego
flat := array.flatten([[1, 2], [3]])
```

Since 1.17.0, `time.parse_duration_ns` accepts days, weeks, and years as well as
the earlier units. `graph.reachable_paths` now returns every reachable path;
policies and tests may see additional paths compared with the incomplete old
result.

## URI parsing

OPA 1.17.0 added `uri.parse` and `uri.is_valid`. `uri.parse` returns RFC 3986
components and omits empty components; possible fields are `scheme`,
`hostname`, `port`, `path`, `raw_path`, `raw_query`, and `fragment`.
`uri.is_valid` returns a boolean for malformed input.

```rego
parsed := uri.parse("https://example.com:8080/api?q=1#top")
valid := uri.is_valid("http://[invalid")
```

## JSON Schema behavior

`json.match_schema` accepts array-rooted documents since 1.13.0. Since 1.17.0,
schema handling supports recursive schemas and `$ref` inside `allOf`, and both
`json.verify_schema` and `json.match_schema` enforce `pattern`. Published schemas
for IR plans and bundle manifests let tooling validate those generated
artifacts.

## Improved negation

OPA 1.17.0 added an opt-in negation behavior. Import `future.keywords.not` so
every compiler-expanded part of a composite expression remains inside the
negated body. With the import, an undefined input or nested call makes `not`
succeed rather than making the containing rule fail. Unlike older
future-keyword imports under Rego v1, this import changes semantics and should
be present whenever a policy uses `not`.

```rego
package example

import future.keywords.not

blocked(name) if startswith(name, "blocked-")

allow if {
    not blocked(input.user)
}
```
