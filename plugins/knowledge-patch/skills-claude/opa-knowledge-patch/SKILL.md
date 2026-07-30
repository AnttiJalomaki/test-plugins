---
name: opa-knowledge-patch
description: Open Policy Agent (OPA)
version: 1.18.0
license: MIT
metadata:
  author: Nevaberry
---


# Open Policy Agent Compatibility Guide

Use this skill when upgrading OPA, migrating Rego, building bundles, embedding
OPA in Go, operating an OPA server, or diagnosing behavior that changed across
recent releases. Check the project's pinned OPA version first and apply only
guidance introduced at or before that version. Prefer the repository's policy,
configuration, tests, manifests, and observed runtime behavior when they
disagree with general guidance.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/migration-and-rego.md](references/migration-and-rego.md) | Rego v1 migration, syntax, compatibility, language changes |
| [references/cli-testing-and-bundles.md](references/cli-testing-and-bundles.md) | CLI streams, formatting, tests, coverage, bundles, Wasm |
| [references/server-plugins-and-security.md](references/server-plugins-and-security.md) | Server binding, REST and Compile APIs, credentials, TLS, security |
| [references/observability-and-decision-logs.md](references/observability-and-decision-logs.md) | Metrics, tracing, logging, decision logs |
| [references/go-sdk-and-tooling.md](references/go-sdk-and-tooling.md) | Go package migration, SDK options, AST tooling, build toolchains |
| [references/evaluation-and-builtins.md](references/evaluation-and-builtins.md) | Evaluation corrections, partial evaluation, built-ins, schemas |

## Migrate Rego v0 to v1

Rego v1 makes `in`, `every`, `if`, and `contains` keywords without
`future.keywords` imports. Use `if` on rules with bodies and `contains` on
multi-value rules:

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

Value assignments still omit `if`. A solitary reference head such as `p.a`
is invalid. Duplicate and shadowing imports are compilation errors, and
`input` and `data` cannot be rule or variable names. Substitution with
`with input as ...` and `with data as ...` remains valid.

Use a current OPA binary to find and rewrite compatibility problems before
linting:

```sh
opa check --v0-v1
opa check --v0-v1 --strict
opa fmt --write --v0-v1
regal lint
```

Removed built-ins include `any`, `all`, `re_match`, `net.cidr_overlap`,
`set_diff`, every `cast_*` conversion, and `cast_null`. See
[references/migration-and-rego.md](references/migration-and-rego.md) for the
complete migration sequence and later syntax changes.

## Upgrade mixed-version bundles safely

Upgrade bundle producers before consumers. Bundles built by OPA v0.64.0 or
later record `rego_version` in their manifest, and that value overrides
`--v1-compatible`.

- While v0 consumers remain, keep policy v0-compatible and run v1 producers
  with `--v0-compatible`, unless every module imports `rego.v1`.
- A v1 consumer loading a bundle from a v0 producer also needs
  `--v0-compatible`; old producers cannot declare the Rego version.
- Recheck mixed-version bundles because per-module version lookup and
  overlapping `file_rego_versions` patterns now resolve deterministically.
- Use `opa check --bundle` to catch overlaps between base JSON/YAML documents
  and virtual documents.

Optimized bundles require an entrypoint with at least package and rule:

```sh
opa build -O=1 -e=authz/allow .
```

See [references/cli-testing-and-bundles.md](references/cli-testing-and-bundles.md)
for bundle metadata, polling validation, Wasm output, and formatter behavior.

## Update Go integrations

Move every OPA import to its `/v1/` package path, including `rego`, `sdk`,
`ast`, `bundle`, `compile`, `types`, and `topdown`:

```go
import "github.com/open-policy-agent/opa/v1/rego"
```

The old paths remain deprecated during OPA 1.0. Register custom built-ins
before concurrent evaluation because `RegisterBuiltin` is not thread-safe.
Direct server integrations must also account for routing through
`http.ServeMux` rather than `gorilla/mux`.

Wasm evaluation now uses the pure-Go wazero runtime, so Wasm-enabled builds no
longer need cgo or a C toolchain. Consult
[references/go-sdk-and-tooling.md](references/go-sdk-and-tooling.md) before
choosing a Go build toolchain or relying on AST source locations, policy-oracle
lookups, evaluation caches, or request hooks.

## Secure and expose the server deliberately

Server mode binds to localhost by default. Bind explicitly when another host
or container must connect:

```sh
opa run --server --addr 0.0.0.0:8181
```

Do not expose vulnerable standalone servers to attacker-controlled Data API
paths. OPA 1.4.0 fixed CVE-2025-46569, under which injected path text could
redirect a request, force success or failure, or consume excessive compute.

Choose fixed point releases when a line has a known follow-up:

- Use 1.4.2 rather than 1.4.1 when tooling needs the versioned capabilities
  file.
- Use at least 1.13.1 for `array.flatten`, and 1.13.2 for the Go security
  rebuild.
- Use 1.16.1 rather than 1.16.0 to avoid the plugin-manager shutdown hang.
- Use 1.17.1 distributed artifacts for the HTTP-handler and crypto-built-in
  standard-library fixes.
- Use 1.18.1 or later for long-running servers to avoid the `AnnotationSet`
  leak, and 1.18.2 before formatting policy.

See
[references/server-plugins-and-security.md](references/server-plugins-and-security.md)
for REST credentials, TLS reloads, compile metadata, runtime limits, and
outbound request identity.

## Treat testing and formatting output as an interface

`opa test` runs in parallel by default, one thread per available CPU core. Use
`--parallel=1` for order-sensitive tests:

```sh
opa test . --parallel=1
```

Parameterized tests can generate named cases from their rule head. Reports
count those cases correctly, JSON results can sort by duration, and test
results now stream one case at a time. Coverage failures include full errors.
Consumers must accept incremental output and rebaseline coverage because
source ranges, inline rule heads, and conjunction expressions are tracked.

`opa eval` writes evaluation errors to stderr. Parse stdout and stderr
separately. Formatting no longer rewrites unchanged files, but use 1.18.2 to
avoid the 1.18.0 single-item collection layout regression.

## Opt in to corrected negation

Import `future.keywords.not` whenever policy uses `not` and the improved
semantics are intended:

```rego
package example

import future.keywords.not

blocked(name) if startswith(name, "blocked-")

allow if {
	not blocked(input.user)
}
```

The import places compiler-expanded parts of a composite expression inside the
negated body. An undefined input or nested call then makes `not` succeed
instead of making the enclosing rule fail. Unlike older future-keyword imports
under Rego v1, this import selects behavior and is not a no-op.

## Recheck corrected evaluation results

Upgrades can intentionally change results. Re-run tests for:

- partial evaluation involving default functions, `every`, nested
  comprehensions, or wall-clock-dependent built-ins;
- Wasm policies using reference-head rules;
- overlapping indexed array and scalar rules;
- `graph.reachable_paths`, which now returns every reachable path;
- wildcard/generated result keys, which no longer synthesize JSON values;
- non-finite `to_number` inputs and overly deep parser input, which now error.

Long-running `regex.replace`, `replace`, `strings.replace_n`, and `concat`
calls now honor cancellation. See
[references/evaluation-and-builtins.md](references/evaluation-and-builtins.md)
for all built-in and partial-evaluation details.

## Configure high-volume operations

For decision logs, the `event` buffer reduces lock contention but gives up the
default buffer's precise memory-footprint guarantee:

```yaml
decision_logs:
  reporting:
    buffer_type: event
    trigger: immediate
```

`trigger: immediate` uploads when a chunk fills; the configured delay remains
the latest upload time. A rotating `file_logger` can receive both server and
decision logs, and rule metadata labels can be merged into `rule_labels`.

Tracing can use HTTP collectors, discovery can join distributed traces, and
Prometheus metrics can be exported through OTLP. See
[references/observability-and-decision-logs.md](references/observability-and-decision-logs.md)
for buffer, masking, metric, trace, logger, and resource-identity details.
