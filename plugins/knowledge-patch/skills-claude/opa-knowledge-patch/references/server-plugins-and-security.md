# Server, Plugins, and Security

## Server exposure and routing

`opa run --server` binds to localhost rather than every interface
(`1.0-migration`). Bind explicitly for access from another host, container, or
remote service:

```sh
opa run --server --addr 0.0.0.0:8181
```

OPA's server uses `http.ServeMux` instead of `gorilla/mux` (since 1.6.0). Go
integrations that directly interface with server routing may need adaptation.

## Data API path-injection fix

OPA 1.4.0 fixes CVE-2025-46569 in earlier standalone servers. When
attacker-controlled text reached a Data API HTTP path, injected Rego could
redirect the requested path, force success or failure, or consume excessive
compute. Risk includes authorization rules that do not exactly match
`input.path` and intermediaries that put unsanitized third-party text in the
path. Upgrade exposed standalone servers.

## REST policy and credential behavior

REST policy uploads honor the runtime Rego version (since 1.0.0), keeping
API-loaded policy aligned with the server's v0 or v1 configuration.

REST plugin authentication additions include:

- AWS SSO credentials as an AWS credential provider (since 1.5.0).
- Azure Key Vault signing of client assertions, keeping signing keys in the
  vault (since 1.5.0).
- Service-account Web Identity credentials for AWS Assume Role signing (since
  1.15.0).

`HTTPAuthPlugin.NewClient()` is called once per `Client` and cached (since
1.15.0). Move per-request counters, transport wrapping, logging, metrics, and
other per-request authentication effects into `Prepare()`; leaving them in
`NewClient()` makes them run only once.

REST TLS configuration exposes `cert_reread_interval_seconds` (since 1.15.0).
The backward-compatible default rereads client certificates on every request.
REST TLS clients also inherit the server's minimum TLS version and cipher
suites.

## Plugin lifecycle

The status plugin supports a graceful-shutdown timeout (since 1.5.0), allowing
shutdown to complete within a configured bound.

OPA 1.16.0 restored bundle-download, `print()`, and other plugin-originated
logs lost in 1.15.x, but could hang in plugin-manager shutdown. Use 1.16.1,
which fixes that regression.

## Compile API SQL filters

The Compile API can translate a Rego query into a PostgreSQL filter (since
1.9.0). Declare unknown data references in document-scoped compile metadata:

```rego
package filters

# METADATA
# scope: document
# compile:
#   unknowns: [input.fruits]
include if input.fruits.name == input.favorite
```

Request the SQL response through the `Accept` header:

```http
POST /v1/compile/filters/include HTTP/1.1
Content-Type: application/json
Accept: application/vnd.opa.sql.postgresql+json

{"input":{"favorite":"pineapple"}}
```

The response `result.query` contains a filter such as
`WHERE fruits.name = E'pineapple'`.

## Custom API metadata

Wrapping servers can read extra top-level request keys through
`BuiltinContext.RequestMetadata`, and custom built-ins can write
`BuiltinContext.ResponseMetadata` (since 1.17.0). Request metadata is logged
under `custom.request_metadata`; non-empty response metadata is returned and
logged. Use namespaced keys to reduce future collisions. The same request and
response metadata plumbing applies to the Compile API handler.

```json
{"input":{"user":"alice"},"com.example.opa/metadata":{"corp-id":"acme-42"}}
```

## Outbound identity and runtime sizing

All outbound OPA HTTP requests use `Open-Policy-Agent/<version>` rather than
the invalid `Open Policy Agent/<version>` product token (since 1.18.0). This
includes bundle, discovery, decision-log, status, `http.send`, and AWS KMS/ECR
requests. Update exact-match log filters and WAF rules:

```text
User-Agent: Open-Policy-Agent/<version> (<os>, <arch>)
```

OPA automatically derives `GOMAXPROCS` from container-aware CPU limits and
`GOMEMLIMIT` from container-aware memory limits (since 1.18.0). Include those
runtime-selected values when sizing or diagnosing container deployments.

## Patched server releases

- OPA 1.13.2 distributed binaries and images use Go 1.25.7, whose standard
  library fixes GO-2026-4337.
- OPA 1.17.1 distributed binaries and images use Go 1.26.4 to fix
  standard-library vulnerabilities exercised by the HTTP handler and crypto
  built-ins. Self-built artifacts depend on their chosen Go version.
- OPA 1.18.1 fixes an `AnnotationSet` memory leak introduced in 1.17.0. Use
  1.18.1 or later for long-running servers.
