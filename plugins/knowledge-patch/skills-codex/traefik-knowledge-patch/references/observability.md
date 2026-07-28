# Observability

Use this reference when configuring metrics, traces, logs, diagnostics, the API,
or the dashboard.

## Identify OTLP telemetry

The OTLP metrics exporter can set OpenTelemetry `service.name` (since 3.2.0).
Set it explicitly when multiple Traefik deployments report to the same
collector so their metric resources remain distinguishable.

Metrics also accept `resourceAttributes` (since 3.5.0). Resource detection now
applies across application logs, access logs, metrics, and traces; when Traefik
runs in Kubernetes, logs and traces automatically gain Kubernetes resource
attributes. Account for those discovered values when writing collector routing
or cardinality rules.

## Correlate access logs and traces

Access logs can record both the trace ID and the entry point's span ID (since
3.2.0). Use the former to join a request to its distributed trace and the
latter to identify the proxy's entry-point span.

Tracing has a verbosity setting and emits fewer spans by default (since 3.5.0).
Configure verbosity explicitly if alerts, tests, or analysis depend on the
older span density.

In 3.7.0, access logs:

- can continue writing to stdio while OTLP log export is active;
- use OpenTelemetry-conformant trace-context attributes; and
- can include Kubernetes Ingress fields.

Update queries that depended on older trace-context attribute names, and enable
the Kubernetes fields when Ingress identity is needed during request analysis.

## Export application and access logs

Application logs and access logs can be exported through OpenTelemetry (since
3.3.0). The OTLP logs integration requires its experimental flag. Configure the
collector and experimental setting together; enabling an exporter without the
gate does not activate this path.

When both a local stream and centralized export are operational requirements,
use the 3.7.0 stdio-plus-OTLP behavior rather than assuming OTLP must replace
stdio.

## Scope observability controls

Metrics, tracing, and access logging can be controlled at entry-point and router
scope (since 3.3.0), not only in global observability configuration. Use the
narrowest scope that expresses policy, and verify inheritance when a router
uses a parent-router hierarchy.

Rejected requests caused by opt-in encoded-character entry-point policies are
written to access logs (since 3.7.0). Include those records when investigating
why a request never reached a router or backend.

ForwardAuth can expose the authenticated identity to access logs through
`LogUserHeader` (since 3.2.0). Treat the selected header as identity-bearing
data and apply the same retention and disclosure policy as other user fields.

## Protect exporter secrets

`metrics.influxdb2.token` can read its secret from a file path (since 3.7.0).
Prefer a mounted secret file when placing the token directly in static
configuration would expose it to configuration distribution or inspection.

## Collect diagnostics through the API

The API provides a support-dump endpoint (since 3.3.0). Use it to capture
diagnostic state, then handle the dump as operationally sensitive material.

The API and dashboard can be mounted under a configurable base path (since
3.3.0). Update router rules, redirects, health checks, and external links when
moving the UI below a prefix.

## Use dashboard details

- The automatic Web UI theme was added and made the default in 3.4.0.
- The certificate overview added in 3.7.0 shows each certificate's domains,
  expiration, and attached HTTP and TCP routers.
- Service details show server weights (since 3.7.0), which helps confirm the
  configured balancing distribution.
- The dashboard name is configurable (since 3.7.0).

Dashboard data is a useful configuration check, but validate request behavior
and emitted telemetry as well; a visible attachment does not prove that a route
or collector works end to end.

## Validate observability

1. Send a request through each affected entry point and router.
2. Confirm access-log, trace, and metric resource identity in the collector.
3. Verify trace ID and entry-point span ID correlation using an actual trace.
4. Check both stdio and OTLP output when both are configured.
5. Exercise the configured API base path and create a support dump.
6. Compare dashboard certificate attachments and server weights with the
   intended dynamic configuration.
