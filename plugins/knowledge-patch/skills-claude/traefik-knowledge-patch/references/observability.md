# Observability

## Metrics identity

From 3.2.0, the OTLP metrics exporter can assign OpenTelemetry `service.name`.
Keep that identity stable and distinct from the applications proxied by the
deployment.

OpenTelemetry metrics add `resourceAttributes` in 3.5.0. Resource detection
also expands across application logs, access logs, metrics, and traces.
Kubernetes resource attributes are added automatically to logs and traces when
running in Kubernetes, so account for those fields in cardinality and storage
policies.

`metrics.influxdb2.token` can read its secret from a file path in 3.7.0,
avoiding a literal token in the static configuration.

## Access-log correlation

Access logs can record the trace ID and the entry point's span ID from 3.2.0.
Use both to connect an incoming proxy request to its distributed trace.

ForwardAuth `LogUserHeader` can add the authenticated identity to logs in
3.2.0; only log a header produced by a trusted authorization service.

In 3.7.0, access logs can remain on stdio while OTLP logging is active. They
use OpenTelemetry-conformant trace-context attributes and can include
Kubernetes Ingress fields.

Rejected requests caused by opt-in encoded-character entry-point policy are
also written to access logs in 3.7.0.

## OpenTelemetry logs

Application-log and access-log OTLP export arrived experimentally in 3.3.0 and
must be enabled with the experimental setting. Validate collector availability
and buffering behavior before depending on it as the only log path.

The later ability to retain stdio output alongside OTLP provides an independent
local collection path.

## Per-route observability

Metrics, tracing, and access logging can be controlled at entry-point and
router scope from 3.3.0 rather than only through global settings. Use the
narrowest scope that still preserves required audit and incident data.

## Trace detail

Tracing gains a verbosity setting in 3.5.0 and emits fewer spans by default.
Set verbosity explicitly if dashboards, alerts, or sampling rules depend on
the previous span detail.

Resource detection in the same release applies to traces and automatically
adds Kubernetes attributes when appropriate.

## Provider diagnostics

Consul, Consul Catalog, and Nomad log their configured provider namespace at
startup from 3.6.0. Use those records to catch namespace mismatches before
investigating service discovery.

The support-dump API, available from 3.3.0, collects diagnostic state for
support and incident analysis. Handle its output as potentially sensitive
operational data.
