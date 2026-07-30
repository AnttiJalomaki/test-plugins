# Observability, cainjector, and Clients

Use this reference for metrics and log migrations, CA bundle injection,
resource discovery, server-side apply clients, and operational queries.

## Metrics migrations

In 1.19, `certmanager_acme_client_request_count` and
`certmanager_acme_client_request_duration_seconds` replaced the high-cardinality
`path` label with the bounded `action` label. Update PromQL, dashboards, and
alerts that query `path`. Recreating the old semantics requires an explicit
Prometheus relabeling or recording rule.

The 1.19 metric `certmanager_certificate_challenge_status` exposes ACME
challenge status.

The 1.18 certificate validity gauges expose issuance and expiration times:

```text
certmanager_certificate_not_before_timestamp_seconds
certmanager_certificate_not_after_timestamp_seconds
```

When Prometheus monitoring is enabled in 1.20, the metrics label is consistently
`cert-manager` rather than varying with the installation namespace or Helm
release name.

In 1.21, chart monitoring uses the fixed `/metrics` path and `http-metrics` port
name. The chart values `prometheus.servicemonitor.targetPort`,
`prometheus.servicemonitor.path`, and `prometheus.podmonitor.path` are removed.
Custom scrape configuration must stop using the old
`tcp-prometheus-servicemonitor` Service port name.

## Structured logs and diagnostic events

The 1.17 logging changes add contextual structured data. Rules that match an
entire line or a literal message string can stop matching; prefer stable fields
and partial predicates.

From 1.20, complete DigitalOcean DNS-01 failures are attached to the Challenge
as events. Inspect those events for provider diagnostics.

## Cainjector bundle handling

`CAInjectorMerging` was opt-in in 1.17 and merged a new CA with the existing
injected bundle rather than replacing it, preserving overlap during issuer
rotation.

It became beta and enabled by default in 1.19. At that point operators could
still explicitly disable the gate to retain replacement semantics.

In 1.21, merging is GA and always enabled; replacement semantics cannot be
restored with a gate. Cainjector also always uses server-side apply, and its
`ServerSideApply` gate is deprecated.

The cainjector `--ignore-namespaces` flag added in 1.21 excludes named
namespaces while watching Secrets for CA injection.

## Resource discovery and selection

`Issuer` and `ClusterIssuer` have the short names `iss` and `ciss` from 1.18:

```console
kubectl get iss
kubectl get ciss
```

From 1.20, the CRDs expose `.spec.issuerRef.group`, `.spec.issuerRef.kind`, and
`.spec.issuerRef.name` as selectable fields:

```console
kubectl get certificates --field-selector spec.issuerRef.name=example-issuer
```

## Client integrations

Generated apply-configuration types for cert-manager resources are available
from 1.19. Go clients can build type-safe server-side apply requests rather than
using unstructured apply payloads.

The deprecated `ObjectReference` API type is removed in 1.21. Integrations that
still compile against or serialize it must migrate.

## Solver resource labels

The 1.21 `--acme-http01-solver-extra-labels` flag lets Helm
`global.commonLabels` propagate to dynamically generated HTTP-01 Pods,
Services, Ingresses, and Gateway API HTTPRoutes. Use it when selectors,
inventory, or metrics depend on common labels.
