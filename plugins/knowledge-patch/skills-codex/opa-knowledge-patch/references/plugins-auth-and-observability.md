# Plugins, authentication, and observability

## Decision-log buffering, masking, and upload

Decision-log masking can address array keys since 1.1.0, allowing values inside
arrays to be redacted.

OPA 1.3.0 adds an event-based buffer for lower lock contention at high request
rates. It does not provide the default buffer's precise memory-footprint
guarantees.

```yaml
decision_logs:
  reporting:
    buffer_type: event
```

Since 1.5.0, adaptive uncompressed-size limits survive decision-log upload
handling, and configuration boundaries derive from `upload_size_limit_bytes`.
Configured upload caps therefore remain in force.

OPA 1.13.0 adds immediate uploads when chunk-size criteria are met. The upload
delay remains the latest an upload can occur.

```yaml
decision_logs:
  reporting:
    trigger: immediate
```

## Logger plugins and rule labels

OPA 1.15.0 introduces logger plugins based on Go's `log/slog.Handler`.
`server.logger_plugin` selects the runtime logger and `decision_logs.plugin`
can send decision logs to the same handler. The built-in `file_logger` writes
rotated structured JSON. Custom builds may register another handler and use
`BufferedLogger` to capture startup output emitted before plugin initialization.

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

Since 1.17.0, metadata annotations accept `labels`. Labels from successful
rules merge with inner-scope precedence (`subpackages` < `package` < `document`
< `rule`), are deduplicated, and appear as top-level `rule_labels`. The runtime
and Go SDK process them by default.

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

## REST authentication and TLS

- Since 1.5.0, REST plugins can obtain AWS credentials from AWS SSO.
- Since 1.5.0, Azure Key Vault can sign REST client assertions so the key need
  not be loaded into OPA.
- Since 1.15.0, AWS request signing can use service-account Web Identity
  credentials when obtaining Assume Role credentials.

OPA 1.15.0 caches `HTTPAuthPlugin.NewClient()` once per `Client`. Move request
counters, transport wrapping, logging, metrics, and all other per-request work
into `Prepare()`; work left in `NewClient()` runs only once.

The same release adds `cert_reread_interval_seconds`. Its backward-compatible
default rereads client certificates on every request. REST TLS configuration
also inherits the server's minimum TLS version and cipher suites.

## JWT verification cache

Since 1.1.0, `io.jwt` token-verification built-ins support a configurable token
cache. Tune memory use against repeated verification work.

## Distributed tracing and OTLP

Since 1.2.0, the discovery plugin participates in distributed tracing, and OPA
server traces accept additional OpenTelemetry resource attributes for
deployment identity.

OPA 1.3.0 supports HTTP collectors when `distributed_tracing.type` is `http`
and exposes finer-grained batch span processor configuration.

```yaml
distributed_tracing:
  type: http
```

Since 1.17.0, distributed-tracing support can export Prometheus metrics through
OTLP to an OTLP collector.

## Metrics

- Since 1.0.0, Prometheus bucket configuration can customize the
  `bundle_loading_duration_ns` histogram.
- Since 1.9.0, Topdown metrics count outbound network requests performed by
  `http.send`.

## Plugin shutdown and diagnostics

Since 1.5.0, the status plugin supports a graceful-shutdown timeout. Set a bound
appropriate to the orchestrator's termination window.

OPA 1.16.0 restores bundle-download, `print()`, and other plugin-originated logs
lost in 1.15.x, but use 1.16.1 because it fixes that release's plugin-manager
shutdown hang.
