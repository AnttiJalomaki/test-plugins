# Connectors and Router Extension Points

## Connector availability and specification selection

Apollo Connectors are generally available in Router 2.0.0. Deployments coming
from Connectors Preview should follow the GA upgrade path.

The Router's default Connector specification changes over time:

- Router 2.14.0 resolves “latest/default” to `connect/v0.3`; schemas explicitly
  linked to `connect/v0.2` remain on v0.2.
- Router 2.16.0 makes an explicit link to
  `https://specs.apollo.dev/connect/v0.4` sufficient. The
  `connectors.preview_connect_v0_4` flag is a deprecated no-op.

Router 2.15.0 deprecates `connectors.subgraphs` in favor of
`connectors.sources`; the old key warns and is scheduled for removal in Router
3.

## Connector transport configuration

### Traffic shaping and TLS

Since 2.1.0, apply Connector traffic shaping to all sources under
`traffic_shaping.connector.all` or to one source under
`traffic_shaping.connector.sources`, keyed as `<subgraph>.<source>`. Connector
traffic shaping does not support `deduplicate_query`. A concrete key has the
shape `subgraph_name.source_name`.

The same key shape applies to custom certificate authorities and mutual TLS
under `tls.connector.sources`.

```yaml
traffic_shaping:
  connector:
    all:
      timeout: 5s
    sources:
      products.inventory:
        global_rate_limit:
          capacity: 20
          interval: 1s
        timeout: 1s
tls:
  connector:
    sources:
      products.inventory:
        certificate_authorities: ${file.ca.crt}
        client_authentication:
          certificate_chain: ${file.client.crt}
          key: ${file.client.key}
```

### Header propagation

Router 2.2.0 lets header rules target all Connectors through
`headers.connector.all` or a source under `headers.connector.sources`, again
keyed `<subgraph>.<source>`. Router YAML overrides headers set by `@connect` or
`@source`.

During the router-v2-migration, body paths used by header propagation must be
rooted JSONPath expressions:

```yaml
headers:
  all:
    request:
      - insert:
          name: from_app_name
          path: $.extensions.metadata[0].app_name
```

### URI mapping and response decoding

Router 2.2.0 permits Connector expressions anywhere in or after the URI path,
including query-parameter names. Expression results stay percent-encoded;
literal `[` and `]` are not encoded unless invalid; trailing slashes survive.
Some placements require Federation 2.11 or later.

Router 2.3.0 decodes Connector response media types as follows:

- A subtype ending in `/json` or `+json` is parsed as JSON.
- `text/plain` becomes a UTF-8 string addressable as `$`.
- Other media types become JSON `null`.
- Missing `Content-Type` still assumes JSON.
- Deserialization failure returns `CONNECTOR_DESERIALIZE` and
  `Response deserialization failed`.

Variables in nested input arguments are accepted by Connector operations from
2.3.0.

## Connector mapping language

### JSON and string operations

Router 2.14.0 adds `->jsonParse`, which parses a JSON string and permits an
immediate selection. A non-string or invalid JSON value fails, and the inferred
parsed shape is unknown.

```text
payload->jsonParse { users { name } }
```

Router 2.15.0 adds:

- `->split(separator[, limit])`; separators can come from data, an empty
  separator splits into UTF-8 characters, and a limit caps results.
- `->trim`, `->trimStart`, and `->trimEnd` for Unicode whitespace.

All reject non-string inputs.

### Connect v0.4 syntax

For schemas linked to `connect/v0.4` in 2.15.0:

- Nested selections accept commas.
- Object-property shorthand is valid.
- Top-level object literals no longer need `$()`.
- A primitive after `name:` is a literal rather than a lookup on `$`.

Qualify intended lookups explicitly, especially REST keys that are not valid
GraphQL names:

```text
{ id, address { street, city }, next: $."@odata.nextLink" }
```

The v0.2 and v0.3 parsers retain their earlier behavior.

Router 2.16.0 fixes v0.4 composition for selections below list-producing arrow
methods such as `->entries`, and stops treating nested scalar-list projections
such as `data->map(@->map(@->toString))` as object group selections.

The separately distributed `connect-migrate` CLI in 2.16.0 compares Connector
selections under their linked specification and v0.4. It classifies
deterministic `$.` rewrites, unchanged selections, and cases requiring review.
It is built from `apollo-federation` with the non-default `connect-migrate`
Cargo feature, not shipped in the Router runtime.

### Requestless mappings

Router 2.15.0 allows `@connect` to omit `http` and resolve from arguments or
enclosing-object data, including inside a nested mutation. Such a mapping
cannot use response body data, `$status`, or `$response`; composition rejects
transport-derived references.

### Recursive input and error merging

Router 2.16.0 lets self-referential Connector input types compose without an
infinite schema walk; expression shape resolution stops a cycle at an unknown
shape.

When a Connector `isSuccess` expression is false, configured
`errors.extensions` deep-merge with defaults in 2.16.0. Supplying a nested
`http` extension therefore retains default members such as `http.status`.

## Connector schema behavior

Router 2.1.2 preserves `@context` and `@fromContext` when introducing a
Connector; earlier affected deployments need that patch level or later.

Router 2.14.0 permits a list-typed argument named by
`@listSize(slicingArguments: [...])` to contribute its length as the demand
multiplier, whether supplied inline or through a variable.

## Coprocessor lifecycle and data selection

### Endpoint and socket selection

Router 2.8.0 lets Router, supergraph, execution, and subgraph stages each set a
`url` overriding the global coprocessor URL. A global-only configuration
continues to work.

Router 2.12.0 supports local coprocessors over Unix domain sockets.

The same release adds `ConnectorRequest` and `ConnectorResponse` stages.
Depending on direction, they can expose URI, headers, body, context, service
identity, and response data:

```yaml
coprocessor:
  url: http://localhost:3007
  connector:
    all:
      request:
        uri: true
        headers: true
        body: true
        context: all
        service_name: true
      response:
        headers: true
        body: true
        context: all
        service_name: true
```

### Response bodies and validation

Router 2.5.0 adds top-level coprocessor `response_validation`, enabled by
default. It also validates subscription termination responses correctly.
Disable only when a coprocessor intentionally returns a shape the Router cannot
validate.

Router 2.2.0 preserves `data: null` when a coprocessor returns a GraphQL
execution error; it no longer drops the data member.

Router 2.14.0 lets supergraph, execution, and subgraph response stages select
`data`, `errors`, and `extensions` separately. Boolean `body` remains valid.
A coprocessor may modify only fields it received; omitted fields keep their
original values.

```yaml
coprocessor:
  supergraph:
    response:
      body:
        data: false
        errors: true
        extensions: true
```

### Header and context safety

Router 2.12.0 makes `externalize_header_map` warn with the invalid non-UTF-8
header name and return the remaining valid headers instead of failing the
whole conversion.

The router-v2-migration namespaces built-in request context keys. Important
replacements include:

```text
apollo_operation_id                           → apollo::supergraph::operation_id
operation_kind                               → apollo::supergraph::operation_kind
operation_name                               → apollo::supergraph::operation_name
apollo_authentication::JWT::claims           → apollo::authentication::jwt_claims
cost.actual                                  → apollo::demand_control::actual_cost
cost.estimated                               → apollo::demand_control::estimated_cost
persisted_query_hit                          → apollo::apq::cache_hit
persisted_query_register                     → apollo::apq::registered
apollo_telemetry::client_name                → apollo::telemetry::client_name
apollo_telemetry::client_version             → apollo::telemetry::client_version
```

Other migration mappings are:

```text
apollo_authorization::authenticated::required → apollo::authorization::authentication_required
apollo_authorization::scopes::required        → apollo::authorization::required_scopes
apollo_authorization::policies::required      → apollo::authorization::required_policies
apollo_override::unresolved_labels            → apollo::progressive_override::unresolved_labels
apollo_override::labels_to_override           → apollo::progressive_override::labels_to_override
apollo_router::supergraph::first_event        → apollo::supergraph::first_event
apollo_telemetry::studio::exclude             → apollo::telemetry::studio_exclude
apollo_telemetry::subgraph_ftv1               → apollo::telemetry::subgraph_ftv1
cost.result                                   → apollo::demand_control::result
cost.strategy                                 → apollo::demand_control::strategy
experimental::expose_query_plan.enabled       → apollo::expose_query_plan::enabled
experimental::expose_query_plan.formatted_plan → apollo::expose_query_plan::formatted_plan
experimental::expose_query_plan.plan          → apollo::expose_query_plan::plan
```

Coprocessors can request `context: deprecated` during migration (`true` was a
deprecated alias), `context: all` for new names, `false` for none, or a
`selective` list. Do not mix selective keys with deprecated names.

Router 2.13.0 fixes the v2.10 regression where `context: true` could delete
returned keys; the `context: deprecated` workaround is no longer needed.

Router 2.16.0 scopes context deletion at parallel subgraph stages: a response
can delete only keys sent to its own stage, not keys concurrently added by
another stage.

Response-stage coprocessor conditions can test
`exists: { request_header: x-name }` from 2.13.0. The Router evaluates and
retains the request-stage fact for response-time use.

## Rhai

Router 2.1.0 exposes `request.uri.scheme` and
`request.subgraph.uri.scheme` as readable and writable, enabling HTTP/HTTPS
rewrites. With `--hot-reload`, Rhai source edits trigger the same reload
mechanism as schema or configuration edits.

Router 2.4.0 preserves a multipart upload's `Content-Type` after Rhai
processing, avoiding the false
`invalid multipart request: Content-Type is not multipart/form-data` failure.

Router 2.14.0 emits `apollo.router.operations.rhai.duration` for every callback.
The histogram is seconds-valued and labels `rhai.stage` and `rhai.succeeded`.

The same release adds `rhai.intern_strings: false`. String interning is enabled
by default; disable it when new-string write-lock contention harms a highly
concurrent workload.

## Rust plugins

### Service lifecycle migration

In the router-v2-migration, a `tower::Service` pipeline is built once and
cloned per request. Do not rely on the construction hook running for every
request. The `cargo-scaffold` generator is removed, though already generated
plugins still compile.

API replacements include:

- `oneshot_checkpoint_async()` → `checkpoint_async()`.
- `OneShotAsyncCheckpointLayer` → `AsyncCheckpointLayer`; call `.buffered()`
  before `.service(...)`.
- `ExtensionsMutex::lock()` → `with_lock()`.
- `TestHarness::build()` → `build_supergraph()`.
- `PluginInit::{new,try_new}()` → `{builder,try_builder}()`.

`services::router::Response::map`, `SchemaSource::File.delay`, and
`ConfigurationSource::File.delay` are removed without listed replacements.
`Context::busy_time`, `Context::enter_active_request`, `BusyTimer`, and
`BusyTimerGuard` are removed because spans already represent processing time.

Subscription request builders regain compatibility in 2.11.0:
`SubscriptionTaskParams` works with `execution::Request` builders, including
unit tests.

### Metrics and OpenTelemetry helpers

The router-v2-migration removes tracing-field metric conversion for fields
prefixed with `counter.`, `histogram.`, `monotonic_counter.`, or `value.` and
logs an error for them. Create OpenTelemetry instruments from
`apollo_router::metrics::meter_provider()`. Gauges, including `.u64_gauge()`,
are exported from 2.1.0.

Router 2.13.0 moves to OpenTelemetry Rust 0.31.0; plugins using unstable direct
APIs must update. In 2.16.0, replace
`apollo_router::otel_compat::{HeaderExtractor, HeaderInjector}` with the
equivalent `opentelemetry_http::{HeaderExtractor, HeaderInjector}` 0.31-or-later
types.
