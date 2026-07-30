---
name: opa-knowledge-patch
description: Open Policy Agent (OPA)
version: 1.18.0
license: MIT
metadata:
  author: Nevaberry
---


# Open Policy Agent Knowledge Patch

Use this skill when writing or migrating Rego, building bundles, embedding OPA,
operating OPA servers, or configuring OPA plugins. Check the project's pinned
OPA version before applying version-dependent behavior. Prefer the manifest,
source, tests, and observed runtime behavior when they disagree with guidance.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/rego-language-and-builtins.md](references/rego-language-and-builtins.md) | Rego v1 syntax, changed evaluation behavior, strings, arrays, schemas, URI and time built-ins |
| [references/cli-testing-and-formatting.md](references/cli-testing-and-formatting.md) | Migration commands, parsing, formatting, tests, coverage, debugger, `eval`, and `exec` |
| [references/bundles-partial-evaluation-and-wasm.md](references/bundles-partial-evaluation-and-wasm.md) | Mixed-version bundles, optimized builds, partial evaluation, Wasm, and capability files |
| [references/go-sdk-ast-and-extensibility.md](references/go-sdk-ast-and-extensibility.md) | Go import paths, evaluation options, AST APIs, policy oracle, custom built-ins, and Compile API |
| [references/server-runtime-and-security.md](references/server-runtime-and-security.md) | Server binding, Data API security, build toolchains, resource limits, outbound headers, and patched releases |
| [references/plugins-auth-and-observability.md](references/plugins-auth-and-observability.md) | REST authentication, decision logs, logger plugins, tracing, metrics, JWT caching, and shutdown |

## Start with breaking changes

### Migrate Rego syntax deliberately

Rego v1 treats `in`, `every`, `if`, and `contains` as keywords. Rules with
bodies need `if`; multi-value rules need `contains`. Value assignments such as
`limit := 10` remain valid without `if`. Solitary reference heads such as
`p.a` are invalid.

```rego
package authz

allow if {
    input.user == "alice"
}

reasons contains "missing role" if {
    not input.role
}

limit := 10
```

Duplicate and shadowing imports are compilation errors. Do not declare rules or
variables named `input` or `data`; document replacement with `with input as`
and `with data as` remains supported. Removed built-ins must be replaced rather
than carried into a v1 policy.

Before changing source, run both compatibility checks, then format in dual-mode:

```sh
opa check --v0-v1
opa check --v0-v1 --strict
opa fmt --write --v0-v1
regal lint
```

### Sequence mixed-version bundle upgrades

Upgrade bundle producers before consumers. Bundles built by OPA v0.64.0 or
later embed `rego_version`, which overrides a consumer's `--v1-compatible`
flag. While v0 consumers remain, keep policy v0-compatible and make v1
producers use `--v0-compatible` unless modules import `rego.v1`. A v1 consumer
loading bundles from an older producer also needs `--v0-compatible`.

For mixed-version bundles, inspect each module's effective version and any
overlapping `file_rego_versions` patterns. OPA 1.18 corrected per-module lookup
and made overlapping patterns deterministic, so revalidate manifests after an
upgrade.

### Bind server interfaces explicitly

`opa run --server` binds to localhost. Services that must be reachable from
another host or container need an explicit address:

```sh
opa run --server --addr 0.0.0.0:8181
```

Treat public binding as a security boundary. OPA 1.4.0 fixed a Data API path
injection vulnerability affecting standalone servers when attacker-controlled
text reached the HTTP path; authorization rules and intermediaries must still
avoid unsanitized path construction.

### Move Go imports to v1 paths

OPA v1 Go packages insert `/v1/` into every OPA import, including `rego`,
`sdk`, `ast`, `bundle`, `compile`, `types`, and `topdown`:

```go
import "github.com/open-policy-agent/opa/v1/rego"
```

Old paths are deprecated. Also audit integrations coupled to OPA server routing:
the server uses `http.ServeMux` rather than `gorilla/mux`.

## High-value release choices

- Use at least 1.4.2 in the 1.4 line. It includes the capability file omitted
  from 1.4.1 as well as the Go security rebuild begun there.
- Use at least 1.13.1 for `array.flatten`; 1.13.0 mishandles single-item arrays.
- Use at least 1.13.2 when relying on distributed binaries or images for the
  Go standard-library fix identified in that rebuild.
- Use 1.16.1 rather than 1.16.0 to avoid the plugin-manager shutdown hang.
- Use 1.17.1 distributed artifacts for their newer Go security rebuild.
- Use at least 1.18.1 for long-running servers because 1.18.0 leaks
  `AnnotationSet` memory.
- Use at least 1.18.2 before formatting policy; it repairs the single-item
  collection layout regression in 1.18.0.

## Rego quick reference

Keyword-named path segments are accepted in dotted references:

```rego
allow if input.package.source == "internal"
```

Primitive numbers cannot have surplus leading zeros. Change `0123` to `123`.
Template strings use a `$` prefix and `{...}` expressions; an undefined
expression renders `<undefined>` instead of making the rule undefined:

```rego
message := $"User {input.username} has role {input.role}"
```

Import `future.keywords.not` whenever a policy needs the improved composite
negation semantics. Unlike older future-keyword imports under Rego v1, this
import selects behavior: compiler-expanded parts stay inside the negated body.

```rego
import future.keywords.not

allow if {
    not blocked(input.user)
}
```

Useful built-ins include `array.flatten`, `uri.parse`, `uri.is_valid`, longer
units in `time.parse_duration_ns`, recursive and pattern-aware JSON Schema
validation, and complete `graph.reachable_paths` results. Consult the language
reference before depending on edge behavior or corrected results.

## CLI and test quick reference

- `opa parse --v0-compatible` parses legacy modules in their intended mode.
- `opa check --bundle ./bundle` detects overlap between base documents and
  virtual documents.
- Optimized bundles require a package-and-rule entrypoint such as
  `opa build -O=1 -e=authz/allow .`.
- `opa test` runs in parallel by default; use `--parallel=1` for serialized
  tests.
- Test results stream case by case. Coverage failures include full errors, and
  coverage source ranges and totals may differ from earlier output.
- `opa eval` writes evaluation errors to stderr; keep stdout and stderr
  handling separate in automation.
- `opa exec` includes the decision ID for correlation with logs and traces.
- The debugger accepts YAML input.

Parameterized tests can generate named cases from their rule heads:

```rego
test_concat[note] if {
    some note, tc in {
        "empty": {"a": [], "b": [], "want": []},
        "filled": {"a": [1], "b": [2], "want": [1, 2]},
    }
    array.concat(tc.a, tc.b) == tc.want
}
```

## Bundle, Wasm, and evaluation quick reference

REST policy uploads use the runtime's configured Rego version. Bundle API
callers receive a default Rego version when they omit one, so do not assume
version metadata remains unset. For Wasm bundles, choose the build option that
retains `print` calls when diagnostics are required.

The Wasm target and SDK use the pure-Go wazero runtime, eliminating cgo and a C
toolchain. Re-test reference-head rules after older planner fixes. Re-run
partial evaluation for policies using default functions, time-dependent
built-ins, nested comprehensions inside `every`, or improved negation inside
`every` because corrected residual policy can change results.

## Go embedding quick reference

Register custom built-ins before concurrent evaluation; `RegisterBuiltin` is
not thread-safe. Embedders can provide a base cache, add map data with
`rego.Data`, select JSON generation per evaluation, wrap `http.send` transports,
and distinguish cancellation from timeout errors.

The policy oracle is public under `v1/ast/oracle`, accepts an existing compiler,
and resolves `some`, `every`, object references, and definitions inside
negation. Preserve AST source locations and do not assume older reference-string
escaping or wildcard-result synthesis.

The Compile API can return PostgreSQL filters when document-scoped metadata
declares unknowns and the request asks for the PostgreSQL SQL media type. Keep
unknown declarations narrow and consume `result.query` as a SQL filter.

## Operations quick reference

OPA 1.18 uses `Open-Policy-Agent/<version>` in outbound `User-Agent` headers.
Update exact-match WAF and log rules. Container-aware runtime setup chooses
`GOMAXPROCS` from CPU limits and `GOMEMLIMIT` from memory limits; include those
automatic values in capacity analysis.

For high-volume decision logs, consider the lower-contention `event` buffer,
`immediate` chunk uploads, rule labels, array-aware masking, and the rotating
`file_logger`. The event buffer sacrifices precise memory-footprint guarantees.
REST authentication work that must happen per request belongs in `Prepare()`,
because `NewClient()` is cached once per client.

## Upgrade validation checklist

1. Identify the exact OPA binary, image, Go module, capability file, and bundle
   producer versions used by each component.
2. Check and format Rego under the intended compatibility mode.
3. Validate bundles for base/virtual document conflicts and effective per-module
   Rego versions.
4. Run tests both normally and with Wasm or partial evaluation when production
   uses those paths.
5. Rebaseline test coverage and any scripts that parse streamed CLI output.
6. Exercise custom built-ins, caches, HTTP transports, metadata, and cancellation
   behavior in embedded evaluators.
7. Verify server binding, Data API path handling, outbound headers, container
   resource limits, and long-running memory use.
8. Test REST credentials, certificate reloads, logging, decision-log upload,
   tracing, metrics export, and bounded plugin shutdown.
