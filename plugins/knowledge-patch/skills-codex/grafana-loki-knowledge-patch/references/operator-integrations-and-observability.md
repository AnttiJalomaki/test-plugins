# Operator, Integrations, and Observability

## Loki Operator identity and OTLP behavior

The Loki Operator supports managed GCP Workload Identity as of 3.4.0. It also
places the log-level attribute in structured metadata. OTLP ingestion adds
`deployment.environment.name` to the default label set.

In 3.5.0, the Operator can:

- drop OTLP attributes;
- configure a TLS CA for Swift; and
- enable time-based stream sharding.

The attribute-dropping behavior is classified as breaking. Review generated
configuration and downstream label or metadata assumptions during an upgrade.

Generated sizing keeps delete workers nonzero and fixes the minimum available
ingesters for the `1x.pico` size (3.5.0). Re-render small installations instead
of preserving values produced by the earlier sizing logic.

## Operator networking, storage, and authorization

As of 3.6.0, the Operator can deploy NetworkPolicies with a LokiStack,
configure virtual-host-style S3 access from Secrets, and apply OpenTelemetry
semantics to LokiStack authorization.

In 3.7.0, the Operator can suppress ingress and customize the gateway server
certificate. Verify routes and trust chains when either control is changed.

Metrics authentication no longer depends on `kube-rbac-proxy` as of 3.7.3.
Remove assumptions about that sidecar from policies, resource sizing, and
monitoring.

## OpenShift behavior

The default OpenShift stream labels change in 3.7.0 as a breaking update.
Review queries, retention, dashboards, and alert rules that use them.

On OCP 4.20 the Operator no longer deploys NetworkPolicies automatically.
Declare the intended policies explicitly. AWS STS deployments receive their
region through an environment variable; ensure that the generated pod has the
expected value.

## OpenTelemetry tracing

Loki uses OpenTelemetry rather than OpenTracing internally as of 3.6.0. It
retains configuration through `JAEGER_`-prefixed environment variables and
exports traces in Jaeger format. Existing operational configuration may remain
valid even though the internal tracing implementation changed.

## Fluent Bit and Fluentd

Fluent Bit v4's `out_loki` plugin can send structured metadata as of 3.6.0.
The Fluentd plugin accepts `compress gzip`. Validate receiver limits using the
uncompressed size even when transport compression is enabled.

## Meta-monitoring migration

Meta-monitoring responsibilities moved from the Grafana meta-monitoring Helm
chart to the Grafana Kubernetes Monitoring Helm chart in 3.6.0. Move release
configuration and ownership to the newer chart and confirm that dashboards,
rules, and telemetry pipelines continue to target the Loki installation.

## Operational UI integration

The Operational UI JavaScript moved to a Grafana plugin in 3.6.0; its server
APIs remain in Loki. Enabling the UI in the Loki chart enables the APIs on
queriers, and the gateway routes UI requests to them. Troubleshoot UI failures
across both the Grafana plugin and the Loki API-routing path.
