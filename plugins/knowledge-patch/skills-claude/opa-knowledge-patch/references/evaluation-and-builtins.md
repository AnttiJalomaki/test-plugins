# Evaluation and Built-ins

## Conversion, units, and HTTP

`units.parse` accepts scientific notation in the numeric component (since
1.0.0), including `units.parse("1e3KB")`.

`to_number` rejects `"Inf"`, `"Infinity"`, and `"NaN"` rather than treating
them as numbers (since 1.0.0).

Topdown HTTP evaluation matches `application/json` content types leniently
(since 1.6.0); a response no longer needs the formerly strict header form to
be treated as JSON.

## Partial evaluation

Partial evaluation has an opt-in to evaluate nondeterministic built-ins (since
1.1.0). The option is available through Topdown, the Rego API, and server
evaluation.

`opa eval --v0-compatible` applies that compatibility mode to
partial-evaluation support modules (since 1.1.0).

Default functions are handled correctly during partial evaluation (since
1.4.0). Recheck residual policy and results for rules that depend on a default
function.

Partial evaluation correctly handles `future.keywords.not` negation inside
`every` and namespaces variables in comprehensions nested inside `every`
(since 1.18.0). Re-run affected queries because residual policy or results can
change.

## JWT and schema built-ins

The `io.jwt` verification built-ins support a configurable token cache (since
1.1.0), trading memory for less repeated verification work.

`json.match_schema` accepts array-rooted documents (since 1.13.0).

JSON Schema support includes recursive schemas, `$ref` inside `allOf`, and
`pattern` validation in `json.verify_schema` and `json.match_schema` (since
1.17.0). Generated schemas are published for IR plans and bundle manifests.

## Arrays, URIs, and durations

`array.flatten` is available from 1.13.0, but that release mishandles
single-item arrays. Use at least 1.13.1:

```rego
flat := array.flatten([[1, 2], [3]])
```

`uri.parse` returns RFC 3986 components, omitting empty components, while
`uri.is_valid` returns a boolean for malformed input (since 1.17.0). Parsed
fields include `scheme`, `hostname`, `port`, `path`, `raw_path`, `raw_query`,
and `fragment`.

```rego
parsed := uri.parse("https://example.com:8080/api?q=1#top")
valid := uri.is_valid("http://[invalid")
```

`time.parse_duration_ns` accepts days, weeks, and years in addition to its
earlier units (since 1.17.0).

## Cancellation and parser bounds

`regex.replace`, `replace`, `strings.replace_n`, and `concat` observe
evaluation context cancellation (since 1.12.0), allowing callers to terminate
long-running operations.

The parser rejects excessive recursion with a parse error (since 1.5.0).
Callers must not assume arbitrary nesting depths are accepted.

## Corrected results

OPA omits generated JSON values for wildcard or generated keys in Rego result
sets (since 1.5.0). Consumers must not expect synthetic values for those keys.

The AST rule index correctly selects candidates for overlapping array and
scalar patterns (since 1.15.0). Re-test policies using those overlaps because
evaluation results can change.

`graph.reachable_paths` returns all reachable paths (since 1.17.0). Policies
and tests that consumed incomplete results can observe additional paths.

## Template error handling

`strings.render_template` no longer emits its former fixed missing-key error
(since 1.13.0). Do not match the old error text when handling absent keys.
