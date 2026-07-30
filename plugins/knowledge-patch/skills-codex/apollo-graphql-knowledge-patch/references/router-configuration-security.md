# Router Configuration, Security, and Deployment

## Router v2 configuration migration

The router-v2-migration stops applying major-version upgrades while loading
configuration. Preview and materialize the upgraded YAML before deployment.
The removed `--schema` flag becomes `router config schema`.

```bash
router config upgrade --diff router.yaml
router config upgrade router.yaml > router.next.yaml
mv router.next.yaml router.yaml
router config schema
```

Minor-version YAML migrations within Router 2 are applied automatically from
2.2.0. Major-version migrations still are not. Regularly materialize and commit
minor migrations so a later major upgrade starts from current configuration.

Router v2 rejects incoming work when busy instead of retaining it in memory.
Stricter traffic shaping can expose more HTTP 503 and 504 responses. During
rollout, monitor CPU and logs and retune timeouts, concurrency, and rate limits.

## Routes and listener behavior

### Supergraph paths

The router-v2-migration changes named path parameters from `:name` to `{name}`;
wildcards must be named and braced:

```yaml
supergraph:
  path: /foo/{bar}/baz
  # Wildcard: /foo/{*rest}
```

Router 2.14.0 normalizes trailing slashes while matching `supergraph.path`, so
`/graphql` also accepts `/graphql/`.

Router 2.3.0 restores the ability to disable the health-check endpoint after
Router 2.0's plugin conversion temporarily lost that behavior.

### Request-header timeouts and limits

Router 2.2.0 adds `server.http.header_read_timeout`; its default is the earlier
hard-coded 10 seconds.

Router 2.9.0 adds `limits.http2_max_headers_list_bytes`, defaulting to 16 KiB.
An oversized HTTP/2 header list is rejected with 431. Since 2.10.0, the limit
applies to TLS, cleartext TCP, and Unix-domain-socket listeners; before that it
covered only TLS.

```yaml
limits:
  http2_max_headers_list_bytes: "48KiB"
```

Known-size GraphQL responses retain `Content-Length` instead of switching to
`transfer-encoding: chunked` from 2.9.0. Body-size hints survive both
client-to-Router and Router-to-subgraph paths.

### GET hardening

Router 2.12.1, in the 2.12.0 batch, rejects a GraphQL `GET` carrying any
`Content-Type` other than `application/json` with optional parameters. It
returns 415. Omitting the header remains allowed subject to CSRF checks. This
is especially important for cookie or HTTP Basic Auth deployments.

### Unix sockets and HTTP/2

Router 2.13.0 supports subgraph endpoints over Unix sockets. Put the request
path in the URL's `path` query parameter:

```text
unix:///tmp/some.sock?path=some_path
```

The same release lets `pool_idle_timeout` control idle keep-alive eviction for
subgraphs, Connector sources, and the coprocessor client. Its default is 15
seconds rather than the previous 5 seconds; `null` disables idle eviction.

For outbound subgraph, Connector, and coprocessor traffic,
`experimental_http2: http2only` uses HTTP/2 prior knowledge over cleartext
connections in 2.13.0. Plain `enable` without TLS still uses HTTP/1.1 because
the client cannot perform h2c upgrade.

Traffic-shaping compression adds `content-encoding`, and every subgraph request
advertises `gzip`, `br`, or `deflate` through `accept-encoding` from 2.11.0.
These headers are added after the request enters the debugging stack, so the
Connectors Debugger does not display them.

Router 2.4.0 fixes a Router 2.3.0 regression that rejected some valid SigV4
configurations at startup and blocked access to SigV4-protected services.

### Response and upload size/time limits

Router 2.15.0 lets `limits.subgraph` and `limits.connector` set global and
per-destination `http_max_response_size`. There is no default. Older
Router-level limit fields migrate under `limits.router`.

An oversized streaming body is stopped and returned as
`SUBREQUEST_HTTP_ERROR`. The Router increments the destination-specific
`apollo.router.limits.*_response_size.exceeded` metric and marks the response
span aborted for `response_size_limit`.

```yaml
limits:
  subgraph:
    all:
      http_max_response_size: 10MB
    subgraphs:
      products:
        http_max_response_size: 20MB
  connector:
    all:
      http_max_response_size: 5MB
    sources:
      products.rest:
        http_max_response_size: 10MB
```

Also in 2.15.0, multipart uploads can bound time spent reading the operations
field with `preview_file_uploads.protocols.multipart.limits.operation_body_timeout`.
The option has no default; expiry returns HTTP 504 and `GATEWAY_TIMEOUT`.

## CORS and Private Network Access

Router 2.5.0 adds ordered `cors.policies` with literal `origins` or regex
`match_origins`. This permits credentials and broader headers only for trusted
origins while retaining a restrictive catch-all.

```yaml
cors:
  policies:
    - match_origins: ["^https://(dev|www)\\.example\\.com$"]
      allow_credentials: true
      allow_headers: ["content-type", "authorization"]
    - origins: ["*"]
      allow_credentials: false
      allow_headers: ["content-type"]
```

Router 2.9.0 lets each policy enable `private_network_access`; `access_id` and
`access_name` are optional.

Invalid CORS values prevent startup in the router-v2-migration instead of being
ignored.

## Authentication and token validation

### Failure policy and context

Router 2.1.0 adds `authentication.router.jwt.on_error`. It defaults to `Error`;
`Continue` ignores JWT-processing errors and leaves claims unset. The outcome
is stored in `apollo::authentication::jwt_status`.

```yaml
authentication:
  router:
    jwt:
      on_error: Continue
```

### Issuers and audiences

Since 2.2.0, each `authentication.router.jwt.jwks` entry accepts an `issuers`
list. Singular `issuer` is auto-migrated during Router 2, but should be removed
before a later major.

Router 2.4.0 adds an optional `audiences` list; a token fails if its `aud`
matches none. Router 2.11.0 accepts `aud` as a string or array and succeeds if
any value matches. Other types, including `null`, fail. `iss` must be a string
or `null`; a string must match configured issuers.

### Expiration and matching keys

Router 2.14.0 lets each JWKS entry use `allow_missing_exp: true`. Missing `exp`
is then accepted, but a supplied expired value is still rejected.

Also in 2.14.0, when several entries share signing key material, issuer or
audience failure on the first signature match no longer stops validation. The
Router tries the other matching entries, supporting shared keys across policy
issuers.

## Authorization and error exposure

The router-v2-migration enables `limits.introspection_max_depth` by default.
Disable it only for a legitimate introspection query that must exceed the
depth; its default is `limits.introspection_max_depth: true`. An empty
`Content-Type` is rejected earlier as possible CSRF with HTTP 400 rather than
415.

Router 2.1.1 fixes resource-exhaustion query vulnerabilities. Earlier releases
need all three mitigations enabled: persisted queries, safelisting, and required
IDs.

```yaml
persisted_queries:
  enabled: true
  safelist:
    enabled: true
    require_id: true
```

Router 2.2.0 makes `include_subgraph_errors` composable. Put global redaction
and extension allowlists under `all`, then refine per subgraph. Per-subgraph
rules can extend or exclude global keys; `deny_extensions_keys` wins over the
global allowlist; `false` redacts everything; an omitted subgraph inherits
`all`. Prefer allowlists because denylists can expose newly introduced fields.

When every requested field is unauthorized, Router 2.13.0 returns `data: null`
and honors `errors.response` (`errors`, `extensions`, or `disabled`) and
`errors.log`, matching partially unauthorized operations.

Client-awareness names and versions supplied through headers or extensions are
validated in 2.13.0. Invalid metadata is rejected.

Router 2.15.0 clarifies that `@authenticated`, `@requiresScopes`, or `@policy`
on a subgraph root type composes onto the shared supergraph root and affects
fields contributed by every subgraph. Put authorization on individual root
fields to keep it scoped.

## Rate and batch limits

Router 2.1.0 adds `batching.maximum_size`. A larger client batch is rejected in
full with HTTP 422 and `BATCH_LIMIT_EXCEEDED`; unset means unlimited.

Rate-limit HTTP semantics changed during Router 2:

- In 2.11.0, enforced rate limits again return HTTP 429 and
  `TOO_MANY_REQUESTS`.
- In 2.13.0, exceeding Router or subgraph rate limits or buffer capacity returns
  HTTP 503 and `SERVICE_UNAVAILABLE`, reverting the 429 behavior. Treat it as
  service load rather than a client-specific throttle.

Configure retries and alerts for the exact deployed version.

Router 2.16.0 adds `limits.router.max_recursive_selections`, with the existing
default of 10,000,000. `limits.router.warn_only: true` makes the check warn
instead of reject. The
`APOLLO_ROUTER_DISABLE_SECURITY_RECURSIVE_SELECTIONS_CHECK` escape hatch
remains available.

Router 2.16.0 also deprecates and ignores
`traffic_shaping.deduplicate_variables`; variable deduplication is always
enabled. Remove the field to clear its startup warning.

## Schema, configuration, and artifact reloads

The router-v2-migration removes `--apollo-uplink-poll-interval` and
`APOLLO_UPLINK_POLL_INTERVAL`. Supergraphs supplied by
`--supergraph-urls` or `APOLLO_ROUTER_SUPERGRAPH_URLS` no longer hot-reload.
Download a remote schema to a local file periodically when reload is required.

Router 2.11.0 polls mutable OCI tag references, including generated variant
tags and custom tags, and reloads when a tag points to a new artifact. For
example, `artifacts.apollographql.com/my-org/my-graph:prod` supports promotion
by retargeting the tag.

Router 2.13.0 allows explicitly safe registry hostnames to serve graph
artifacts over HTTP, supporting private registries and trusted pull-through
caches.

Router 2.15.0 retries transient schema or related reload failures instead of
permanently retaining the previous schema. `reload.max_retries` defaults to 5;
`0` disables retries and `null` allows unlimited attempts. `retry_delay`
defaults to 10 seconds. Any new reload trigger resets the retry budget.

```yaml
reload:
  max_retries: 5
  retry_delay: 10s
```

Router release downloads can use a remote proxy mirror from 2.1.0 when direct
GitHub access is unavailable.

## Packaging and Helm

Router 2.7.0 adds Helm `deploymentAnnotations` for annotations on the
Deployment. Keep `podAnnotations` for pod annotations.

In 2.10.0, the DIY Dockerfile pins its Rust builder to a Bookworm variant such
as `rust:1.91.1-slim-bookworm`, so the builder and Bookworm runtime use
compatible glibc. A generic new Rust image can otherwise build a binary that
fails with `GLIBC_2.39 not found`.

Router 2.13.0 derives Helm `ServiceMonitor.metadata.name` from the
`router.fullname` helper, honoring `nameOverride` and `fullnameOverride`. With
defaults, a `my-release` release changes from `my-release` to
`my-release-router`.

Restricted features are blocked even when the license is only in a warning
state from 2.11.0. The Router returns an error instead of continuing to use the
feature.
