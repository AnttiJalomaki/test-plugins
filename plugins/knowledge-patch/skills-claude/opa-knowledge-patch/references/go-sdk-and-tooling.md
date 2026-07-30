# Go SDK and Tooling

## Package-path migration

OPA v1 inserts `/v1/` into Go import paths (`1.0-migration`). Migrate every OPA
dependency, including `rego`, `sdk`, `ast`, `bundle`, `compile`, `types`, and
`topdown`:

```go
import "github.com/open-policy-agent/opa/v1/rego"
```

Old package paths are deprecated but remain for the lifetime of OPA 1.0.

## Build toolchains

Use the toolchain required by the OPA line being built:

| OPA release | Go toolchain |
| --- | --- |
| 1.0.0 | Go 1.22 or newer |
| 1.9.0 | Go 1.25.1 |
| 1.12.0 | Go 1.25.5 |
| 1.13.2 distributed artifacts | Go 1.25.7 security rebuild |
| 1.15.0 | Go 1.26.1 |
| 1.17.1 distributed artifacts | Go 1.26.4 security rebuild |

Self-built 1.17 artifacts inherit the security properties of the selected Go
version.

## HTTP and evaluation hooks

Customize the transport used by `http.send` with eval-level
`EvalHTTPRoundTrip` or query-level `WithHTTPRoundTrip` (since 1.0.0). The hook
wraps Topdown's configured `http.Transport` and returns an
`http.RoundTripper`.

Evaluation errors distinguish a canceled context from a timeout (since
1.0.0), allowing callers to react to the real termination reason.

The bundle plugin trigger method supports direct error handling (since 1.0.0),
so integrations can handle trigger failures at the call site.

Supply a caller-owned base cache to Rego or Topdown evaluation (since 1.2.0).

Pass map-backed data directly with the `rego.Data` option (since 1.13.0):

```go
r := rego.New(
	rego.Query("data.authz.allow"),
	rego.Data(map[string]any{"roles": []any{"admin"}}),
)
```

Provide Rego's `GenerateJSON` function per evaluation (since 1.17.0), allowing
calls through one integration to make separate JSON-generation choices.

Register custom built-ins before evaluations or other concurrent registry
users begin. `RegisterBuiltin` is not thread-safe (since 1.6.0).

## AST conversion and source fidelity

`ast.InterfaceToValue` accepts `[]string` and existing `ast.Value` instances
directly (since 1.2.0).

AST reference-to-string conversion uses a JSON-escaped literal only when
necessary (since 1.5.0). Tools comparing serialized reference strings can see
less escaping.

Reference resolution preserves `Location` on `SomeDecl` nodes (since 1.5.0),
letting AST-based tools retain source positions.

Inner `ast.Not` expressions carry source locations (since 1.18.0). The policy
oracle can find definitions inside them, removing the prior negation blind
spot for editors and analyzers.

## Policy oracle

The public oracle lives at
`github.com/open-policy-agent/opa/v1/ast/oracle`, and callers can provide an
existing compiler (since 1.2.0).

Oracle support includes `some` and `every`, and `FindDefinition` resolves
object references (since 1.6.0).

## Partial-run and server integration

Topdown `PartialRun()` initializes wall-clock time correctly (since 1.4.0).
Embedded partial evaluations using wall-clock-dependent built-ins can return
corrected results after upgrading.

The server routes through `http.ServeMux` rather than `gorilla/mux` (since
1.6.0). Direct routing integrations may require changes.

User-supplied `commit` and `timestamp` values in version information are
preserved (since 1.5.0), allowing custom builds to retain injected provenance.

## Wasm runtime

The `wasm` evaluation target and WASM SDK use the pure-Go wazero runtime (since
1.18.0) rather than `wasmtime-go`. Wasm-enabled builds no longer require cgo
or a C compiler.
