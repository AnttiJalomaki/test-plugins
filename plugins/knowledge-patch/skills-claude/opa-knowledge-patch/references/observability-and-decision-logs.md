# Observability and Decision Logs

## Metrics

Prometheus configuration accepts custom histogram buckets for
`bundle_loading_duration_ns` (since 1.0.0), letting operators choose useful
ranges for slow bundle loads.

Topdown metrics count outbound network requests made by the `http.send`
built-in (since 1.9.0).

Distributed tracing can export Prometheus metrics through OTLP (since 1.17.0),
so an OTLP collector can receive OPA runtime metrics.

## Tracing

Discovery participates in distributed tracing, and server tracing accepts
additional OpenTelemetry resource attributes (since 1.2.0). Use the attributes
to attach deployment-specific resource identity.

Distributed tracing supports HTTP collectors when
`distributed_tracing.type: http` and exposes finer-grained batch span
processor configuration (since 1.3.0):

```yaml
distributed_tracing:
  type: http
```

## Decision identification and masking

`opa exec` includes the decision ID in its output (since 1.2.0), enabling
correlation with decision logs and traces.

Decision-log masking can address array keys (since 1.1.0), so sensitive
elements within arrays can be removed before upload.

## Buffers and upload timing

Set `decision_logs.reporting.buffer_type` to `event` for a lower-contention
event-based buffer (since 1.3.0):

```yaml
decision_logs:
  reporting:
    buffer_type: event
```

The event buffer improves high-load lock contention, but does not offer the
default buffer's precise memory-footprint guarantee.

Decision-log uploads preserve their adaptive uncompressed-size limit, and the
plugin derives configuration boundaries from `upload_size_limit_bytes` (since
1.5.0). Configured upload caps remain effective throughout upload handling.

Set `decision_logs.reporting.trigger` to `immediate` to upload as soon as the
configured chunk-size criteria are met (since 1.13.0). The configured delay is
still the latest time an upload occurs:

```yaml
decision_logs:
  reporting:
    trigger: immediate
```

## Rotating server and decision logs

Logger plugins implement Go's `log/slog.Handler` (since 1.15.0). Select a
runtime logger with `server.logger_plugin` and point `decision_logs.plugin` to
the same plugin to share output. The built-in `file_logger` writes rotating
structured JSON:

```yaml
server:
  logger_plugin: file_logger

decision_logs:
  plugin: file_logger

plugins:
  file_logger:
    path: /var/log/opa/server.log
    max_size_mb: 100
    max_age_days: 28
    max_backups: 3
    compress: true
    level: info
```

Custom builds can register another handler. Use `BufferedLogger` to retain
startup logs emitted before plugin initialization.

## Rule labels

Metadata annotations accept `labels` (since 1.17.0). Labels from every
successfully evaluated rule merge with inner-scope precedence:
`subpackages` < `package` < `document` < `rule`. OPA deduplicates them and
emits the result as the top-level `rule_labels` array. The runtime and Go SDK
process these annotations by default.

```rego
# METADATA
# scope: package
# labels:
#   service: authz
#   severity: info
package myapp

# METADATA
# labels:
#   severity: low
#   team: platform
allow if input.role == "admin"
```

The resulting labels include `service: authz`, `severity: low`, and
`team: platform`.
