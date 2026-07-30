# Go SDK, AST, and extensibility

## Import paths and toolchain baseline

For the `1.0-migration`, every OPA Go import moves under `/v1/`, including
`rego`, `sdk`, `ast`, `bundle`, `compile`, `types`, and `topdown`. Old paths are
deprecated but remain for the lifetime of OPA 1.0.

```go
import "github.com/open-policy-agent/opa/v1/rego"
```

OPA 1.0.0 requires Go 1.22 or newer when built from source or as part of a Go
integration. See the runtime reference for later release-specific build
toolchains.

## Evaluation options and caches

Since 1.0.0, an embedded evaluator can customize requests from `http.send` by
using eval-level `EvalHTTPRoundTrip` or query-level `WithHTTPRoundTrip`. The
option wraps the Topdown-configured `http.Transport` and returns an
`http.RoundTripper`.

Cancellation and timeout errors are distinct since 1.0.0, so integrations can
react to the actual termination cause. Since 1.2.0, Rego and Topdown evaluations
can use a caller-provided base cache.

OPA 1.13.0 adds `rego.Data`, allowing callers to provide map data without first
constructing a store:

```go
r := rego.New(
    rego.Query("data.authz.allow"),
    rego.Data(map[string]any{"roles": []any{"admin"}}),
)
```

Since 1.17.0, a caller can provide the Rego `GenerateJSON` function per
evaluation, so evaluations through one integration can choose different JSON
generation behavior.

## AST conversion and source fidelity

`ast.InterfaceToValue` accepts `[]string` and an existing `ast.Value` directly
since 1.2.0. Since 1.5.0, compiler reference resolution retains `Location` on
`SomeDecl` nodes. The same release changes AST reference-to-string conversion
to use JSON-escaped literals only when required, so serialized strings can
contain less escaping.

OPA 1.18.0 attaches source locations to inner `ast.Not` expressions, enabling
editors and analyzers to inspect definitions inside negation without a location
blind spot.

## Policy oracle

OPA 1.2.0 makes the oracle public at
`github.com/open-policy-agent/opa/v1/ast/oracle` and lets callers supply an
existing compiler. Since 1.6.0, it understands `some` and `every`, and
`FindDefinition` resolves object references. Since 1.18.0, it can find
definitions for expressions inside inner `ast.Not` nodes.

## Results and rule indexing

Since 1.5.0, OPA does not synthesize JSON values for wildcard or generated keys
in result sets. Consumers must not expect fabricated values for those keys.

OPA 1.15.0 fixes candidate selection for indexed rules with overlapping array
and scalar patterns. Re-test policies that depend on those overlaps because
evaluation results can change.

## Custom built-ins and HTTP evaluation

Complete custom built-in registration before starting concurrent evaluations:
`RegisterBuiltin` is not thread-safe as of 1.6.0.

Also since 1.6.0, Topdown accepts lenient forms of the `application/json`
`Content-Type` for HTTP responses instead of requiring the formerly strict
header representation.

## Server integration and routing

OPA 1.6.0 replaces `gorilla/mux` with `http.ServeMux` for server routing. Go
integrations that interact directly with the router must adapt rather than
assuming gorilla-specific behavior.

## Compile API to PostgreSQL

OPA 1.9.0 can compile a Rego query to a PostgreSQL filter. Declare unknown data
references in document-scoped compile metadata, then request the PostgreSQL SQL
response media type.

```rego
package filters

# METADATA
# scope: document
# compile:
#   unknowns: [input.fruits]
include if input.fruits.name == input.favorite
```

```http
POST /v1/compile/filters/include HTTP/1.1
Content-Type: application/json
Accept: application/vnd.opa.sql.postgresql+json

{"input":{"favorite":"pineapple"}}
```

The response returns a filter in `result.query`, such as
`WHERE fruits.name = E'pineapple'`.

## Custom request and response metadata

Since 1.17.0, wrapping servers can read extra top-level request fields from
`BuiltinContext.RequestMetadata`, while custom built-ins can populate
`BuiltinContext.ResponseMetadata`. Request metadata is logged under
`custom.request_metadata`; non-empty response metadata is returned to the
caller and logged. Use namespaced keys to avoid collisions with future fields.
The Compile API handler supports the same metadata path.

```json
{"input":{"user":"alice"},"com.example.opa/metadata":{"corp-id":"acme-42"}}
```

## Version information

Since 1.5.0, the runtime preserves caller-supplied `commit` and `timestamp`
fields in version information, allowing custom builds to keep injected
provenance.
