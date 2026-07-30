# Router Telemetry and Observability

## Router v2 metric migration

### Removed metrics and replacements

During the router-v2-migration, move dashboards and alerts from removed
Router-specific metrics:

| Removed metric | Replacement |
| --- | --- |
| `apollo_router_http_request_retry_total` | `http.client.request.duration` with `http.request.resend_count`; set `default_requirement_level: recommended` |
| `apollo_router_timeout` | Status 504 on `http.server.request.duration` or `http.client.request.duration` |
| `apollo_router_http_requests_total`, `apollo_router_http_request_duration_seconds` | `http.server.request.duration` and `http.client.request.duration` |
| `apollo_router_session_count_total` | `apollo.router.open_connections` from Router 2.1.0 |
| `apollo_router_session_count_active` | `http.server.active_requests` |
| `apollo_require_authentication_failure_count` | Server duration with status 401 |
| `apollo_authentication_failure_count`, `apollo_authentication_success_count` | `apollo.router.operations.authentication.jwt`, using `authentication.jwt.failed` |
| `apollo_router_deduplicated_subscriptions_total` | `apollo.router.operations.subscriptions` with `subscriptions.deduplicated` |
| Cache hit/miss counts | Derive from `apollo.router.cache.hit.time` and `apollo.router.cache.miss.time` |

`apollo_router_span` and `apollo_router_processing_time` have no direct
replacement. Request spans expose `busy_ns` and `idle_ns` for a synthetic
overhead calculation.

### Dotted metric names

The same migration renames remaining underscore metrics:

```text
apollo_router_opened_subscriptions          → apollo.router.opened.subscriptions
apollo_router_cache_hit_time                → apollo.router.cache.hit.time
apollo_router_cache_size                    → apollo.router.cache.size
apollo_router_cache_miss_time               → apollo.router.cache.miss.time
apollo_router_state_change_total            → apollo.router.state.change.total
apollo_router_span_lru_size                 → apollo.router.exporter.span.lru.size
apollo_router_uplink_fetch_count_total      → apollo.router.uplink.fetch.count.total
apollo_router_uplink_fetch_duration_seconds → apollo.router.uplink.fetch.duration.seconds
```

### Default telemetry behavior

The router-v2-migration changes defaults:

- `telemetry.instrumentation.spans.mode`: `spec_compliant`.
- `telemetry.apollo.signature_normalization_algorithm`: `enhanced`.
- `telemetry.apollo.metrics_reference_mode`: `extended`.
- GraphOS usage reporting through OTLP is enabled under
  `otlp_tracing_sampler`; replace the pre-v1.61
  `experimental_otlp_tracing_sampler` name.

## Attributes, selectors, and conditions

### Static and dynamic metric attributes

In the router-v2-migration, move static attributes from
`telemetry.exporters.metrics.common.attributes` to `common.resource`. Put
dynamic selectors on individual instruments under
`telemetry.instrumentation.instruments`.

```yaml
telemetry:
  instrumentation:
    instruments:
      router:
        http.server.request.duration:
          attributes:
            content_type:
              request_header: content-type
              default: application/json
  exporters:
    metrics:
      common:
        resource:
          environment: production
```

### Response selectors

The removed `subgraph_response_body` selector is split during the
router-v2-migration:

- `subgraph_response_data`, whose JSONPath root is the response `data`.
- `subgraph_response_errors`, whose root is the errors array.

Router 2.3.0 adds `response_body` for a complete Router response. Prefer the
more targeted `response_errors` added in 2.7.0 when only GraphQL errors are
needed; its JSONPath root is the response errors array. For example, a request
to `/graphql?operation=value` records `http.route` as `/graphql`.

When a plugin on the Router service changes
`apollo::telemetry::client_name` or `apollo::telemetry::client_version`, Router
2.3.0 propagates the updated values to spans and traces.

Router 2.16.0 adds:

- `response_errors_count`, counting JSONPath matches over the error array.
- `response_errors_field`, evaluating a JSONPath per error and returning the
  matches as a string array.

Router 2.9.0 adds `response_cache_control` for computed subgraph
`Cache-Control` values—for example, `max_age` can feed a seconds histogram—and
documents `active_subgraph_requests`.

Router 2.14.0 adds a Router-service `request_duration` selector. It returns
floating-point seconds or integer milliseconds/nanoseconds and can drive
attributes or conditions.

### Connector selectors

Router 2.6.0 adds Connector instrument selectors for:

- `supergraph_operation_name`
- `supergraph_operation_kind`
- a named `request_context`
- `connector_on_response_error`

`connector_on_response_error` is true when `is_successful` fails, or when the
status is non-200 if no success condition exists. The existing
`connector_request_mapping_problems` and
`connector_response_mapping_problems` selectors also accept a boolean form
that reports whether any problem exists.

### Conditions and standard attributes

The router-v2-migration replaces
`telemetry.exporters.logging.experimental_when_header` with conditions under
`telemetry.instrumentation.events` on Router, supergraph, or subgraph
request/response events. A subgraph condition reads the original client header
with `supergraph_request_header`.

Router 2.13.0 allows response-stage events to test
`exists: { request_header: x-name }` with `on: response`; the request-stage
fact is retained for response evaluation.

Router 2.14.0 lets Router spans alias `client.name`, `client.version`,
`http.route`, and `http.request.method` without changing their default
emission. Router 2.16.0 lets the standard `client.name` and `client.version`
metric attributes use selectors again, not only booleans or aliases.

`http.route` contains only the matched path, not the query string, from 2.3.0.

## Span placement and HTTP attributes

Router 2.8.0 lets the `http_client` span record request headers inserted by
Rhai with a `request_header` selector.

Router 2.12.0 moves attributes configured under
`telemetry.instrumentation.spans.http_client` from `subgraph_request` to the
`http_request` span. Update searches and processors.

Router 2.13.0 prohibits conditions and the `static` selector on `http_client`
span attributes; either causes startup failure.

Every outbound `http_request` span carries `http.response.status_code` from
2.11.0. An unsuccessful response also carries `error.type`.

Router 2.13.0 adds `context_id: true` at Router, supergraph, subgraph, and
Connector instrumentation stages. It exposes the unique request context ID,
also available to Rhai as `request.id`, for correlating spans, logs, and custom
events.

Router 2.13.0 also makes `http.client.response.body.size` and
`http.server.response.body.size` consistently report compressed bytes when
compression is used, even without `Content-Length`, across client, subgraph,
and Connector responses.

Router 2.15.0 adds `apollo.router.connection.acquire.duration` for a newly
created TCP or Unix connection to a subgraph, Connector, or coprocessor. Pool
hits do not record it. Attribute with `network.transport` and the applicable
`subgraph.name`, `connector.source.name`, or coprocessor identity.

The response-size limit counters introduced in 2.15.0 are
`apollo.router.limits.subgraph_response_size.exceeded` and
`apollo.router.limits.connector_response_size.exceeded`.

## Metrics and events by subsystem

### Pipelines, memory, and allocator

Router 2.1.0 adds `apollo.router.pipelines`, labeled by `schema.id`, optional
`launch.id`, and `config.hash`. It reveals old pipelines held by long requests
or subscriptions after reload.

Linux builds using the default `global-allocator` feature expose jemalloc
metrics from 2.5.0:

```text
apollo_router_jemalloc_active
apollo_router_jemalloc_allocated
apollo_router_jemalloc_mapped
apollo_router_jemalloc_metadata
apollo_router_jemalloc_resident
apollo_router_jemalloc_retained
```

Use them to distinguish application allocations, active pages, allocator
metadata, resident memory, mapped extents, and virtual mappings retained
instead of returned to the OS.

Request and query-planner allocation histograms arrive in 2.11.0. See the
planning reference for their build constraints and attributes.

### Errors

Connector and demand-control error spans carry GraphQL error-code span events
from 2.1.0. The extended-error preview option is
`telemetry.apollo.errors.preview_extended_error_metrics`, replacing
`telemetry.apollo.errors.experimental_otlp_error_metrics`.

Extended metrics respect each subgraph's `send` setting. A per-subgraph
`telemetry.apollo.errors.subgraph.[all|(subgraph name)].redaction_policy` can
be `ErrorRedactionPolicy.Strict` or
`ErrorRedactionPolicy.Extended`; with `redact: true`, extended policy permits
`extensions.code` to reach Studio.

From 2.1.0, value-completion failures increment
`apollo.router.graphql.error` and `apollo.router.operations.error` with
`RESPONSE_VALIDATION_FAILED`, even though those failures do not appear in the
GraphQL error array.

Router 2.3.0 includes original message and path for Connector and demand-control
errors in GraphOS traces. Router 2.7.0 adds service attribution for `_entities`
fetch failures so Studio identifies the responsible subgraph or Connector.

With `preview_extended_error_metrics: enabled`, Router 2.16.0 emits span events
carrying `graphql.error.extensions.code` for counted subgraph, supergraph,
execution, parse, and validation errors, extending Connector and demand-control
coverage.
The complete option is
`telemetry.apollo.errors.preview_extended_error_metrics: enabled`.

### Cardinality

Router 2.1.0 increments
`apollo.router.telemetry.metrics.cardinality_overflow` when a batch exceeds
2,000 attribute combinations and OpenTelemetry ignores excess attributes.

Router 2.16.0 adds a common `cardinality_limit` and per-view override.
Overflow combinations collapse into a series with
`otel_metric_overflow="true"`. Raising a limit consumes more memory; monitor
the overflow counter.
The paths are `telemetry.exporters.metrics.common.cardinality_limit` and
`views[].cardinality_limit`.

```yaml
telemetry:
  exporters:
    metrics:
      common:
        cardinality_limit: 5000
        views:
          - name: http.server.request.duration
            cardinality_limit: 20000
```

A view without `aggregation` now preserves a counter or gauge's native
aggregation in 2.16.0. Configure histogram aggregation explicitly when
dashboards require `_bucket`, `_sum`, and `_count`.

### Duration and overhead

Custom duration instruments convert values to configured `s`, `ms`, `us`, or
`ns` from 2.8.0 rather than always recording seconds. Seconds remain the
recommended default.

The same release adds opt-in `apollo.router.overhead`, measuring Router work
excluding subgraph and Connector waits; coprocessor request time is still
included. It also permits exported metric renaming through OpenTelemetry views.

Router 2.14.0 adds the seconds-valued
`apollo.router.operations.rhai.duration`, labeled by callback stage and
success.

Default Router histogram buckets range from 0.001 to 10 seconds. From 2.11.0
guidance, configure `telemetry.exporters.metrics.common.buckets` to cover
timeouts above ten seconds or long values collapse into the highest boundary.

### State and subscriptions

Router 2.9.0 removes `_redacted` suffixes from event names on
`apollo.router.state.change.total`; `UpdateLicense` still appends license
state. For example, `updateconfiguration_redacted` becomes
`updateconfiguration`.

Subscription counters and termination reasons evolve in 2.9.0 and 2.14.0.
See the subscription reference for exact event-count exclusions and end-reason
values.

## Apollo and third-party export

### Batch processors

Router 2.1.0 uses
`telemetry.apollo.batch_processor.scheduled_delay` for the secondary,
high-cardinality real-time metrics path; other Apollo metrics retain a fixed
60-second interval. `telemetry.apollo.batch_processor.max_export_timeout` also
controls the Apollo OTLP metrics `PeriodicReader`.

Router 2.7.0 adds destination-specific processors:

- `tracing.batch_processor` for Apollo OTLP and usage-report traces.
- `metrics.otlp.batch_processor` for Apollo OTLP metrics.
- `metrics.usage_reports.batch_processor` for usage-report metrics.

Legacy `telemetry.apollo.batch_processor` remains the fallback. OTLP metrics
`scheduled_delay` does not apply to configuration-gauge metrics.

### Subgraph Insights flag progression

Router 2.6.0 introduces an Apollo-only, non-customizable subgraph fetch-duration
histogram under `telemetry.apollo.experimental_subgraph_metrics`. Third-party
OpenTelemetry exporters do not receive it; use
`http.client.request.duration` for a customizable equivalent.

Rename the option to `preview_subgraph_metrics` in 2.7.0 and to the generally
available `subgraph_metrics` in 2.8.0.

### Resource attributes and Prometheus

The Prometheus exporter publishes resources only through `target_info` by
default. From 2.4.0, set `resource_selector: all` to copy configured resource
attributes onto every Prometheus metric. This does not affect OTLP.

### Export protocols, proxies, and environment variables

The router-v2-migration removes the dedicated Jaeger exporter. Keep `jaeger`
propagation if needed, but export via OTLP to a collector with OTLP enabled on
4317 for gRPC or 4318 for HTTP.

Router 2.13.0 deprecates the native Zipkin exporter and removes its service-name
setting. Use Zipkin's OTLP endpoint.

GraphOS OTLP over HTTP honors `HTTP_PROXY`, `HTTPS_PROXY`, and `NO_PROXY` from
2.14.0. A TLS-inspecting proxy also requires its root certificate in the Router
trust store.

Router 2.14.0 offers experimental Apollo HTTP transport through
`telemetry.apollo.experimental_otlp_metrics_protocol` and
`telemetry.apollo.experimental_otlp_tracing_protocol`; gRPC remains preferred.

OTLP endpoint environment handling changes materially:

- Router 2.4.0 honors `OTEL_EXPORTER_OTLP_ENDPOINT`,
  `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT`, and
  `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`, and corrects the default HTTP metrics
  endpoint path. Unencrypted values can log exporter errors even while
  delivery succeeds.
- Router 2.11.0 warns that generic `OTEL_EXPORTER_OTLP_ENDPOINT` overrides
  defaults and can redirect traces away from Studio.
- Router 2.13.0 refuses to start when any of those three variables is set.
  Remove inherited definitions.

### Sampling

Router 2.16.0 lets OTLP, Zipkin, Datadog, and Apollo tracing exporters set
independent absolute sampling fractions. An exporter fraction cannot exceed
`telemetry.exporters.tracing.common.sampler`. Datadog ignores its field when
agent sampling is enabled.

## Logging and data protection

Router 2.14.0 adds `expand_json_string_values: true` for stdout and file JSON
formatters. A string containing a valid JSON object or array is emitted as
native JSON for backend indexing. OTLP export is unchanged.

Router 2.16.0 masks sensitive header values in logs, telemetry, coprocessor
communication, and Apollo trace-header forwarding even with no explicit
`masking` block. The built-in header list is case-insensitive.

Configured global and per-subgraph lists are additive unless
`replace_defaults: true`; Connectors inherit their parent subgraph's rules.
A telemetry header selector can choose `redact: mask` or `redact: allow`.
The shared `http_client` layer applies only global rules. A secret copied into
a coprocessor body or context is not protected by header masking.

## Anonymous telemetry and fleet detection

From 2.5.0, `APOLLO_TELEMETRY_DISABLED` disables anonymous telemetry only. It
does not disable identifiable metrics from the fleet detector plugin; do not
treat it as a global telemetry switch.
