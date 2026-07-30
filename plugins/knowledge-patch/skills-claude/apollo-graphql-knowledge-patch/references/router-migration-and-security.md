# Apollo Router migration, configuration, and security

## Router v2 migration (`router-v2-migration`)

### Materialize configuration upgrades

Router v2 does not apply major-version migrations while loading configuration.
Preview and materialize the upgraded YAML before deployment. The removed
`--schema` flag is replaced by `router config schema`.

```sh
router config upgrade --diff router.yaml
router config upgrade router.yaml > router.next.yaml
mv router.next.yaml router.yaml
router config schema
```

Within Router 2, minor-version YAML migrations are applied automatically
(`2.2.0`). Still materialize and commit them regularly; automatic migration does
not cross the next major boundary.

### Backpressure

The v2 architecture rejects work when busy instead of retaining it in memory.
Stricter traffic shaping can expose more `503` and `504` responses. Monitor CPU and
logs while retuning timeouts, concurrency, and rate limits.

### Route and selector syntax

Named supergraph parameters use `{name}`, not `:name`; wildcards are braced and
named:

```yaml
supergraph:
  path: /foo/{bar}/baz
  # path: /foo/{*rest}
```

Header-propagation body paths require a `$` JSONPath root:

```yaml
headers:
  all:
    request:
      - insert:
          name: from_app_name
          path: $.extensions.metadata[0].app_name
```

From `2.14.0`, trailing slashes are normalized when matching `supergraph.path`, so
`/graphql` and `/graphql/` both match a configured `/graphql`.

### Remote supergraphs

`--apollo-uplink-poll-interval` and `APOLLO_UPLINK_POLL_INTERVAL` are removed.
Supergraphs supplied through `--supergraph-urls` or
`APOLLO_ROUTER_SUPERGRAPH_URLS` do not hot-reload. Download remote schemas to a
local file on a schedule when reload behavior is needed.

### Plugin lifecycle and API changes

The `cargo-scaffold` generator is gone, although generated plugins continue to
compile. A `tower::Service` pipeline is built once and cloned per request; do not
assume a construction hook runs per request.

- `oneshot_checkpoint_async()` becomes `checkpoint_async()`.
- `OneShotAsyncCheckpointLayer` becomes `AsyncCheckpointLayer`; call
  `.buffered()` before `.service(...)`.
- `ExtensionsMutex::lock()` becomes `with_lock()`.
- `TestHarness::build()` becomes `build_supergraph()`.
- `PluginInit::{new,try_new}()` becomes `{builder,try_builder}()`.
- `services::router::Response::map`, `SchemaSource::File.delay`, and
  `ConfigurationSource::File.delay` are removed without listed replacements.
- `Context::busy_time`, `Context::enter_active_request`, `BusyTimer`, and
  `BusyTimerGuard` are removed; spans already represent processing duration.

```rust
AsyncCheckpointLayer::new(move |request: execution::Request| {
    // ...
})
.buffered()
.service(service)
.boxed()
```

External plugins can again use `SubscriptionTaskParams` with
`execution::Request` builders, including tests (`2.11.0`).

Router `2.13.0` adopts OpenTelemetry Rust 0.31.0; plugins using unstable upstream
APIs must update. In `2.16.0`, replace
`apollo_router::otel_compat::{HeaderExtractor, HeaderInjector}` with identical
`opentelemetry_http::{HeaderExtractor, HeaderInjector}` 0.31+ types.

### Namespaced request context

Update plugins, Rhai, coprocessors, and selectors:

```text
apollo_authentication::JWT::claims            → apollo::authentication::jwt_claims
apollo_authorization::authenticated::required → apollo::authorization::authentication_required
apollo_authorization::scopes::required        → apollo::authorization::required_scopes
apollo_authorization::policies::required      → apollo::authorization::required_policies
apollo_operation_id                           → apollo::supergraph::operation_id
apollo_override::unresolved_labels            → apollo::progressive_override::unresolved_labels
apollo_override::labels_to_override           → apollo::progressive_override::labels_to_override
apollo_router::supergraph::first_event         → apollo::supergraph::first_event
apollo_telemetry::client_name                 → apollo::telemetry::client_name
apollo_telemetry::client_version              → apollo::telemetry::client_version
apollo_telemetry::studio::exclude             → apollo::telemetry::studio_exclude
apollo_telemetry::subgraph_ftv1               → apollo::telemetry::subgraph_ftv1
cost.actual                                   → apollo::demand_control::actual_cost
cost.estimated                                → apollo::demand_control::estimated_cost
cost.result                                   → apollo::demand_control::result
cost.strategy                                 → apollo::demand_control::strategy
experimental::expose_query_plan.enabled       → apollo::expose_query_plan::enabled
experimental::expose_query_plan.formatted_plan→ apollo::expose_query_plan::formatted_plan
experimental::expose_query_plan.plan          → apollo::expose_query_plan::plan
operation_kind                                → apollo::supergraph::operation_kind
operation_name                                → apollo::supergraph::operation_name
persisted_query_hit                           → apollo::apq::cache_hit
persisted_query_register                      → apollo::apq::registered
```

Coprocessors can temporarily request `context: deprecated` (`true` is a deprecated
alias), request new names with `context: all`, omit with `false`, or use a
`selective` list. Selective keys cannot be mixed with deprecated names.

## Authentication and JWT

### Failure policy (`2.1.0`)

`authentication.router.jwt.on_error` defaults to `Error`. `Continue` ignores JWT
processing failures and leaves claims unset. The context records the outcome in
`apollo::authentication::jwt_status`.

```yaml
authentication:
  router:
    jwt:
      on_error: Continue
```

### Issuers, audiences, and expiry

- These policies are configured per `authentication.router.jwt.jwks` entry.
- A `jwks` entry accepts `issuers` (`2.2.0`). Singular `issuer` is auto-migrated
  during Router 2 but should be upgraded before Router 3.
- A `jwks` entry accepts `audiences` (`2.4.0`); the token must match at least one.
- As of `2.11.0`, `aud` may be a string or string array. `null` and other types
  fail. `iss` must be a string or `null`, and a string must match configured
  issuers.
- `allow_missing_exp: true` is per JWKS entry (`2.14.0`). A supplied expiry is
  still enforced.
- When several entries share key material, Router `2.14.0` continues trying
  matching entries after an issuer or audience mismatch, supporting reused RSA
  keys such as Azure AD B2C policies.

```yaml
authentication:
  router:
    jwt:
      jwks:
        - url: https://example.com/.well-known/jwks.json
          issuers: [https://issuer.one, https://issuer.two]
          audiences: [https://my.api]
          allow_missing_exp: true
```

## CORS and private networks

Router `2.5.0` adds ordered `cors.policies` with literal `origins` or regular
expression `match_origins`, enabling different credentials and headers per origin.

```yaml
cors:
  policies:
    - origins: ["https://studio.apollographql.com"]
    - match_origins: ["^https://(dev|staging|www)?\\.my-app\\.com$"]
      allow_credentials: true
      allow_headers: ["content-type", "authorization"]
    - origins: ["*"]
      allow_credentials: false
      allow_headers: ["content-type"]
```

In `2.9.0`, a policy can enable Private Network Access through
`private_network_access`; `access_id` and `access_name` are optional.

Invalid CORS values stop Router v2 startup instead of being ignored.

## Request hardening and limits

### Batches and headers

`batching.maximum_size` rejects an oversized entire client batch with HTTP 422 and
`BATCH_LIMIT_EXCEEDED`; unset means unlimited (`2.1.0`).

`server.http.header_read_timeout` controls header-read time and defaults to ten
seconds (`2.2.0`).

`limits.http2_max_headers_list_bytes` defaults to 16 KiB and rejects oversized
HTTP/2 header lists with 431 (`2.9.0`). From `2.10.0`, it covers TLS, cleartext
TCP, and Unix-socket listeners.

### GET content types

Router `2.12.1` (the `2.12.0` batch) rejects GraphQL GET requests with a
`Content-Type` other than `application/json` plus optional parameters, returning
415. Omitting it remains valid subject to CSRF checks. This is security-sensitive
for cookie and HTTP Basic authentication.

An empty `Content-Type` is rejected early as possible CSRF with HTTP 400 under the
v2 security defaults, rather than 415.

### Response sizes and uploads (`2.15.0`)

`limits.subgraph` and `limits.connector` set global and per-destination
`http_max_response_size`; no default exists. Old Router-level limit fields migrate
under `limits.router`. An oversized streaming body stops with
`SUBREQUEST_HTTP_ERROR`, increments the corresponding subgraph or connector
response-size exceeded metric, and marks the response span aborted for
`response_size_limit`.

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
```

Multipart file uploads can independently bound reading of the operations field.
`operation_body_timeout` has no default; expiration returns 504 with
`GATEWAY_TIMEOUT`.

```yaml
preview_file_uploads:
  enabled: true
  protocols:
    multipart:
      enabled: true
      limits:
        operation_body_timeout: 5s
```

The response-size counters are
`apollo.router.limits.subgraph_response_size.exceeded` and
`apollo.router.limits.connector_response_size.exceeded`.

### Recursive selections (`2.16.0`)

`limits.router.max_recursive_selections` sets the fragment-expansion ceiling,
default 10,000,000. `limits.router.warn_only: true` warns instead of rejects. The
`APOLLO_ROUTER_DISABLE_SECURITY_RECURSIVE_SELECTIONS_CHECK` escape hatch remains.

## Validation, authorization, and redaction

Router v2 enables `limits.introspection_max_depth` by default. Disable it only for
a legitimate deeper introspection query; its default is
`limits.introspection_max_depth: true`.

Router `2.1.1` fixes resource-exhaustion query vulnerabilities. Earlier releases
need persisted queries, safelisting, and required IDs together as mitigation:

```yaml
persisted_queries:
  enabled: true
  safelist:
    enabled: true
    require_id: true
```

`include_subgraph_errors` in `2.2.0` supports global redaction and extension
allowlists with per-subgraph refinements. Per-subgraph entries extend or exclude
global keys; `deny_extensions_keys` wins over the allowlist; `false` redacts all;
omission inherits `all`. Prefer allowlists to avoid exposing future fields.

`supergraph.redact_query_validation_errors: true` replaces every validation
failure with `invalid query` and code `UNKNOWN_ERROR` (`2.12.0`).

Router `2.12.0` validates unknown fields inside input-object variables. Set
`supergraph.strict_variable_validation: measure` only to preserve non-enforcing
migration behavior.

When all fields are unauthorized, Router `2.13.0` returns `data: null` and honors
the configured `errors.response` destination and `errors.log`.

Applying `@authenticated`, `@requiresScopes`, or `@policy` to a subgraph root type
composes it onto the shared supergraph root (`2.15.0`). Use field directives when
policy should affect only one subgraph's contribution.

## Sensitive headers and client metadata

Router `2.16.0` masks sensitive header values in logs, telemetry, coprocessor
communications, and Apollo trace-header forwarding even without a `masking` block.
Built-in, global, and per-subgraph case-insensitive lists are additive unless
`replace_defaults: true`; Connectors inherit the parent subgraph policy.

Telemetry selectors can override with `redact: mask` or `redact: allow`. The shared
`http_client` layer applies only global rules. Secrets copied into coprocessor body
or context fields are not automatically masked there.

Enhanced client-awareness names and versions are validated from `2.13.0`; invalid
header or extension values are rejected.

## Building custom Router images (`2.10.0`)

The DIY Dockerfile pins its Rust builder to a Bookworm variant such as
`rust:1.91.1-slim-bookworm`, matching the Bookworm runtime glibc. A generic Rust
builder may select a newer glibc and produce `GLIBC_2.39 not found` at startup.
