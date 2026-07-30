# Apollo Router observability and telemetry

## Router v2 OpenTelemetry migration (`router-v2-migration`)

### Removed metric replacements

Migrate dashboards and alerts as follows:

- `apollo_router_http_request_retry_total` becomes
  `http.client.request.duration` with `http.request.resend_count`; set
  `default_requirement_level: recommended`.
- `apollo_router_timeout` becomes status 504 on
  `http.server.request.duration` or `http.client.request.duration`.
- `apollo_router_http_requests_total` and
  `apollo_router_http_request_duration_seconds` become server and client request
  duration instruments.
- `apollo_router_session_count_total` becomes
  `apollo.router.open_connections` (available from `2.1.0`);
  `apollo_router_session_count_active` becomes `http.server.active_requests`.
- `apollo_require_authentication_failure_count` becomes server duration with 401.
- `apollo_authentication_failure_count` and
  `apollo_authentication_success_count` become
  `apollo.router.operations.authentication.jwt`, distinguished by the
  presence/value of `authentication.jwt.failed`.
- `apollo_router_deduplicated_subscriptions_total` becomes
  `apollo.router.operations.subscriptions` with `subscriptions.deduplicated`.
- Derive cache hit/miss counts from `apollo.router.cache.hit.time` and
  `apollo.router.cache.miss.time`.
- `apollo_router_span` and `apollo_router_processing_time` have no direct
  replacement. Request spans expose `busy_ns` and `idle_ns` for synthesized
  overhead.

Remaining Router metrics use dotted names:

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

### Defaults

`telemetry.instrumentation.spans.mode` defaults to `spec_compliant`,
`telemetry.apollo.signature_normalization_algorithm` to `enhanced`, and
`telemetry.apollo.metrics_reference_mode` to `extended`.

GraphOS operation usage through OTLP is enabled under `otlp_tracing_sampler`.
Replace the pre-v1.61 `experimental_otlp_tracing_sampler` name.

### Selector and attribute migration

Removed `subgraph_response_body` becomes `subgraph_response_data` or
`subgraph_response_errors`; each payload part is its own JSONPath root.

```yaml
attributes:
  value:
    subgraph_response_data: $.test
  error_code:
    subgraph_response_errors: $[*].extensions.extra_code
```

Static metric attributes move from
`telemetry.exporters.metrics.common.attributes` to `common.resource`. Dynamic
attributes belong on instruments.

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
          env_full_name: deployment_env
```

Conditional logging moves from removed
`telemetry.exporters.logging.experimental_when_header` to conditions on router,
supergraph, or subgraph `telemetry.instrumentation.events`. At subgraph stages,
read the original client header with `supergraph_request_header`. Dynamic metric
configuration belongs under `telemetry.instrumentation.instruments`.

### Plugin metrics

The Router no longer converts `tracing` fields prefixed `counter.`,
`histogram.`, `monotonic_counter.`, or `value.` into metrics and logs an error for
them. Rust plugins must obtain OpenTelemetry instruments from
`apollo_router::metrics::meter_provider()`. Plugin gauges, including
`.u64_gauge()`, export correctly from `2.1.0`.

### Jaeger and Zipkin

The dedicated Jaeger exporter is removed. Keep Jaeger propagation if necessary,
but export over OTLP to collector ports 4317 (gRPC) or 4318 (HTTP).

```yaml
telemetry:
  exporters:
    tracing:
      propagation:
        jaeger: true
      otlp:
        enabled: true
```

Native Zipkin export is deprecated in `2.13.0` and cannot set a service name; use
Zipkin's OTLP endpoint.

## Core metrics and resource visibility

### Cardinality

`apollo.router.telemetry.metrics.cardinality_overflow` increments when a batch
exceeds the default 2,000 attribute combinations and excess attributes are
ignored (`2.1.0`).

Router `2.16.0` adds
`telemetry.exporters.metrics.common.cardinality_limit` and per-view
`views[].cardinality_limit`. Overflow combinations collapse into a series with
`otel_metric_overflow="true"`. Raising limits increases memory usage.

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

A view without `aggregation` now preserves the instrument's native counter or
gauge aggregation. Specify histogram aggregation when existing dashboards require
`_bucket`, `_sum`, and `_count`.

### Active pipelines (`2.1.0`)

`apollo.router.pipelines` counts active request pipelines by `schema.id`, optional
`launch.id`, and `config.hash`, revealing old pipelines held by long operations
after reload.

### Allocator and request memory

Linux builds using default `global-allocator` expose jemalloc active, allocated,
mapped, metadata, resident, and retained metrics (`2.5.0`):

```text
apollo_router_jemalloc_active
apollo_router_jemalloc_allocated
apollo_router_jemalloc_mapped
apollo_router_jemalloc_metadata
apollo_router_jemalloc_resident
apollo_router_jemalloc_retained
```

`apollo.router.request.memory` covers full-request allocations and
`apollo.router.query_planner.memory` planning jobs (`2.11.0`). Both expose
`allocation.type` and `context`, and require Unix, `global-allocator` enabled, and
`dhat-heap` disabled.

### Router overhead and duration

Enable `apollo.router.overhead` to measure Router processing excluding subgraph and
Connector waits (`2.8.0`); coprocessor request time is currently included.

The router `request_duration` selector measures elapsed time from arrival
(`2.14.0`). Units are float seconds or integer milliseconds/nanoseconds and can
drive attributes or conditions.

### Connection acquisition (`2.15.0`)

`apollo.router.connection.acquire.duration` records new TCP or Unix connection
setup to a subgraph, Connector, or coprocessor. Pool hits are not recorded. Use
`network.transport` plus `subgraph.name`, `connector.source.name`, or `coprocessor`
for attribution.

## HTTP and span attributes

`http.route` contains only the matched path, never the query string, from `2.3.0`;
`/graphql?operation=value` records `/graphql`.

Every outbound `http_request` span carries `http.response.status_code` from
`2.11.0`; failures also carry `error.type`.

Attributes configured under `telemetry.instrumentation.spans.http_client` attach
to the `http_request` span, not `subgraph_request`, from `2.12.0`.

That section does not support conditions or a `static` selector (`2.13.0`);
configuring either prevents startup.

The `http_client` span can record request headers inserted by Rhai (`2.8.0`):

```yaml
telemetry:
  instrumentation:
    spans:
      mode: spec_compliant
      http_client:
        attributes:
          http.request.header.some_rhai_header:
            request_header: some_rhai_header
```

Router `2.13.0` makes `http.client.response.body.size` and
`http.server.response.body.size` report compressed bytes consistently for client,
subgraph, and Connector responses, including when `Content-Length` is absent.

Standard router-span attributes `client.name`, `client.version`, `http.route`, and
`http.request.method` can be aliased from `2.14.0`. Default emission is unchanged.
In `2.16.0`, `client.name` and `client.version` metric attributes can again use
selectors, not just booleans or aliases.

## Errors and response selectors

### Extended error metrics

Rename `telemetry.apollo.errors.experimental_otlp_error_metrics` to
`preview_extended_error_metrics` (`2.1.0`). Extended metrics honor each subgraph's
`send` value.
`telemetry.apollo.errors.subgraph.[all|(subgraph name)].redaction_policy` can be
`ErrorRedactionPolicy.Strict` or `ErrorRedactionPolicy.Extended`; with
`redact: true`, Extended permits `extensions.code` to reach Studio.

With `telemetry.apollo.errors.preview_extended_error_metrics: enabled`, counted
Connector and demand-control errors already emit code events. Router `2.16.0` extends
`graphql.error.extensions.code` span events to counted subgraph, supergraph,
execution, parse, and validation errors.

Value-completion failures count as `RESPONSE_VALIDATION_FAILED`; Connector and
demand-control spans carry their corresponding GraphQL codes.

### Response bodies and errors

`response_body: true` captures a Router response body from `2.3.0`.

Prefer `response_errors` from `2.7.0` when only GraphQL errors are needed; its
JSONPath root is the response error array.

```yaml
response_errors: "$.[0].message"
```

Router `2.16.0` adds `response_errors_count`, counting matches from a JSONPath, and
`response_errors_field`, evaluating a path per error and returning a string array.

Entity-fetch errors include responsible service attribution from `2.7.0`.

### Cache and active-request selectors

`response_cache_control` exposes computed subgraph `Cache-Control` values
(`2.9.0`), for example `max_age` as a seconds histogram.

The `active_subgraph_requests` attribute selector is documented and available from
`2.9.0`.

Connector instruments can select `supergraph_operation_name`,
`supergraph_operation_kind`, named `request_context`, and
`connector_on_response_error` (`2.6.0`). The last is true when `is_successful`
fails, or status is non-200 when no condition exists.
`connector_request_mapping_problems` and
`connector_response_mapping_problems` also accept a boolean “has any problem”
form.

## Metrics configuration and exports

### Prometheus resources (`2.4.0`)

Prometheus exports resources through `target_info` by default. Set
`resource_selector: all` to attach resource attributes to every Prometheus metric;
OTLP is unaffected.

### Units and views

Duration instruments convert recorded values to configured `s`, `ms`, `us`, or
`ns` from `2.8.0`; seconds remain recommended. OpenTelemetry views can rename
Router instruments.

Default histogram buckets span 0.001–10.0 seconds (`2.11.0`). Configure
`telemetry.exporters.metrics.common.buckets` to cover longer timeouts or long
observations will accumulate at the top boundary.

### OpenTelemetry endpoint environment variables

Router `2.4.0` began honoring `OTEL_EXPORTER_OTLP_ENDPOINT`,
`OTEL_EXPORTER_OTLP_METRICS_ENDPOINT`, and
`OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`, with occasional spurious errors on
unencrypted endpoints despite successful delivery.

By `2.11.0`, the generic endpoint could redirect Studio export and caused a startup
warning. Router `2.13.0` refuses to start when any of those endpoint variables is
set. Remove inherited values and use Router configuration.

### Proxy and HTTP transport

GraphOS OTLP over HTTP respects `HTTP_PROXY`, `HTTPS_PROXY`, and `NO_PROXY` from
`2.14.0`; TLS inspection requires the proxy root in Router trust.

Experimental Apollo telemetry HTTP transport is controlled by
`telemetry.apollo.experimental_otlp_metrics_protocol` and
`telemetry.apollo.experimental_otlp_tracing_protocol`; gRPC remains preferred.

### Per-exporter sampling (`2.16.0`)

OTLP, Zipkin, Datadog, and Apollo exporters accept absolute `sampler` fractions.
An exporter cannot exceed `telemetry.exporters.tracing.common.sampler`; Datadog's
setting is ignored when agent sampling is on.

```yaml
telemetry:
  exporters:
    tracing:
      common:
        sampler: 0.1
      otlp:
        enabled: true
        sampler: 0.02
```

## Apollo and GraphOS reporting

### Batch processors

Realtime high-cardinality metrics use a secondary delivery path whose interval
follows `telemetry.apollo.batch_processor.scheduled_delay`; other Apollo metrics
keep 60 seconds (`2.1.0`).
`telemetry.apollo.batch_processor.max_export_timeout` also controls the Apollo OTLP
metrics `PeriodicReader`.

Router `2.7.0` allows destination-specific settings:

- `tracing.batch_processor` for Apollo OTLP and usage-report traces.
- `metrics.otlp.batch_processor` for Apollo OTLP metrics.
- `metrics.usage_reports.batch_processor` for usage-report metrics.

Old `telemetry.apollo.batch_processor` values remain fallbacks. OTLP metrics
`scheduled_delay` does not affect configuration-gauge metrics.

### Subgraph Insights flag progression

The GraphOS-only fetch-duration histogram began as
`telemetry.apollo.experimental_subgraph_metrics` (`2.6.0`), became
`preview_subgraph_metrics` (`2.7.0`), then GA `subgraph_metrics` (`2.8.0`).
It cannot be customized or exported to third-party backends; use
`http.client.request.duration` for a customizable equivalent.

### Anonymous telemetry and fleet detection

`APOLLO_TELEMETRY_DISABLED` disables anonymous telemetry only (`2.5.0`); it does
not disable identifiable fleet-detector metrics.

## Redis cache metrics (`2.6.0`)

Stable query-plan-cache metrics include:

- `apollo.router.cache.redis.connections` initially, replaced by
  `apollo.router.cache.redis.clients` in `2.8.0`.
- `apollo.router.cache.redis.command_queue_length`.
- `apollo.router.cache.redis.commands_executed`.
- `apollo.router.cache.redis.redelivery_count`.
- `apollo.router.cache.redis.errors`, classified by type.

Experimental averages are
`experimental.apollo.router.cache.redis.network_latency_avg`,
`experimental.apollo.router.cache.redis.latency_avg`,
`experimental.apollo.router.cache.redis.request_size_avg`, and
`experimental.apollo.router.cache.redis.response_size_avg`; their names or
behavior may change. Redis `metrics_interval` controls collection and defaults to
one second.

## Operation, state, and streaming metrics

`apollo.router.opened.subscriptions` includes `graphql.operation.name` from
`2.4.0`.

Names in `apollo.router.state.change.total` lose their `_redacted` suffix in
`2.9.0` (for example, `updateconfiguration_redacted` becomes
`updateconfiguration`); `UpdateLicense` still appends license state.

`apollo.router.operations.subscriptions.events` counts data events but excludes
ping, pong, and close from `2.9.0`.

`apollo.router.operations.recursion` and
`apollo.router.operations.lexical_tokens` expose parser complexity from `2.12.0`.

`context_id: true` adds the unique request ID to router, supergraph, subgraph, and
Connector instrumentation (`2.13.0`); Rhai exposes it as `request.id`.

```yaml
telemetry:
  instrumentation:
    spans:
      router:
        attributes:
          request.id:
            context_id: true
```

Streaming termination reason attributes and counters introduced in `2.14.0` are
listed in `router-execution-and-delivery.md`.

## Logging

For stdout and file JSON formatters, `expand_json_string_values: true` emits string
attributes containing valid JSON arrays or objects as native JSON (`2.14.0`).
OTLP exporters are unaffected.

Router-service plugins that update `apollo::telemetry::client_name` or
`apollo::telemetry::client_version` now affect spans and traces (`2.3.0`).
