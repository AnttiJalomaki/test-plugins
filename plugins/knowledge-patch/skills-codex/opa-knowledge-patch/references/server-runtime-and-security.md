# Server runtime and security

## Network binding

Since the `1.0-migration`, `opa run --server` binds to localhost rather than
every interface. Explicitly opt in when the process must be reachable from a
host, peer container, or remote service:

```sh
opa run --server --addr 0.0.0.0:8181
```

Review firewalling and authentication whenever exposing that listener.

## Data API path injection

OPA 1.4.0 fixes CVE-2025-46569 in earlier standalone servers. An attacker who
could place controlled text into a Data API HTTP path could inject Rego that
redirected the requested path, forced success or failure, or consumed excessive
compute. Exposure included authorization policy that did not exactly match
`input.path` and intermediaries that copied unsanitized third-party text into
the path. Upgrade and keep path construction constrained.

## Runtime Rego mode

Since 1.0.0, policies uploaded through the RESTful policy API use the runtime's
configured Rego version. Upload clients should not assume API-loaded policy is
parsed independently of the server's v0 or v1 mode.

## Release build toolchains

When reproducing official builds from source, select the release's required Go
toolchain:

| OPA release | Go toolchain guidance |
| --- | --- |
| 1.0.0 | Go 1.22 or newer |
| 1.9.0 | Go 1.25.1 |
| 1.12.0 | Go 1.25.5, bumped from 1.25.4 |
| 1.13.2 | Go 1.25.7 security rebuild for GO-2026-4337 |
| 1.15.0 | Go 1.26.1 |
| 1.17.1 | Go 1.26.4 security rebuild for HTTP-handler and crypto-built-in exposures |

Self-built artifact security depends on the selected Go version, not merely the
OPA tag.

## Patched point releases

- OPA 1.4.1 rebuilds with Go 1.24.2 for CVE-2025-22870 and CVE-2025-22871 but
  omits `capabilities/v1.4.1.json`. Use 1.4.2 for the complete patched 1.4 line.
- OPA 1.13.2 distributed binaries and images contain the Go 1.25.7 standard
  library fix; use at least that point release when relying on distributed
  artifacts.
- OPA 1.16.0 restores bundle-download, `print()`, and other plugin-originated
  logs dropped by 1.15.x, but its plugin manager can hang at shutdown. OPA
  1.16.1 fixes the hang.
- OPA 1.17.1 distributed binaries and images contain the Go 1.26.4 security
  rebuild. A source build's posture follows its actual toolchain.
- OPA 1.18.1 fixes an `AnnotationSet` memory leak introduced in 1.17.0. Use it
  or later for long-running servers, especially when diagnosing excess memory.

## Outbound HTTP identity

OPA 1.18.0 changes the `User-Agent` for every outbound HTTP path—bundle,
discovery, decision log, status, `http.send`, and AWS KMS/ECR—from the invalid
space-separated product name to:

```text
User-Agent: Open-Policy-Agent/<version> (<os>, <arch>)
```

Update exact-match WAF rules, proxies, metrics, and log filters.

## Container resource limits

OPA 1.18.0 again derives `GOMAXPROCS` automatically from container-aware CPU
limits and also derives `GOMEMLIMIT` from container-aware memory limits.
Capacity planning and incident diagnosis must account for those runtime-selected
values rather than assuming host-wide Go defaults.

## Wasm runtime dependencies

OPA 1.18.0 moves the Wasm evaluator and SDK to the pure-Go wazero runtime. A
Wasm-enabled OPA build no longer needs cgo or a C compiler toolchain.
