# Upgrades and deprecations

## Agent migration

- Promtail is deprecated in 3.4.0 because its code moved into Grafana Alloy.
  Alloy supplies migration guidance and a configuration-conversion utility.
  Lambda-promtail is explicitly outside this deprecation.
- Promtail is removed as of 3.7.3. Move Promtail configurations to Alloy;
  Lambda-promtail remains available.
- The Promtail Docker image no longer contains `wget` (3.4.0). This breaks
  probes, scripts, and derived images that assumed the binary was present.

## Storage, configuration, and chart lifecycle

- BoltDB storage, additional legacy configuration options, and legacy API
  endpoints are deprecated in 3.4.0. Inventory each before upgrading.
- Deprecated ksonnet configurations are removed in 3.5.0; this is a breaking
  cleanup rather than an alias or automatic migration.
- Simple Scalable Deployment mode is deprecated in 3.6.0 and scheduled for
  removal before Loki 4.0.
- The community `LGTM-distributed`, `loki-canary`, `loki-distributed`, and
  `loki-simple-scalable` Helm charts are deprecated in 3.6.0.
- Effective March 16, 2026, the open-source chart associated with 3.7.0 moved
  to `grafana-community/helm-charts` for community maintenance. Update chart
  source references and automation. The GEL chart remains separately
  maintained.
- Meta-monitoring responsibilities moved in 3.6.0 from the Grafana
  meta-monitoring Helm chart to the Grafana Kubernetes Monitoring Helm chart.

## Breaking ingestion and label behavior

- Operator support for dropping OTLP attributes is classified as breaking in
  3.5.0. Review Operator-generated ingestion behavior and retained attributes.
- Parsed labels no longer override same-named structured metadata (3.7.0).
  Audit parser pipelines and queries that depend on collision precedence.
- The Operator's default OpenShift stream labels change in 3.7.0. Treat this
  as a breaking label-set change affecting selectors, cardinality, retention,
  and dashboards.

## Breaking query execution

- Scheduler capacity accounting now uses total compute capacity (3.7.0).
- Worker threads are shared across all scheduler connections (3.7.0).

Both scheduler changes are classified as breaking. Re-test fairness,
concurrency, saturation, and capacity calculations together.

## Deployment compatibility

- Helm object-store values use `object_store.storage_prefix` rather than
  `object_store.prefix` as of 3.5.0.
- The nginx service no longer receives a ServiceMonitor in 3.5.0. Move any
  scraping expectation to the intended monitoring resources.
- Loki Dockerfiles set `/` as the working directory in 3.7.0. Make relative
  paths explicit in scripts and derived images.
- On OpenShift Container Platform 4.20, the Operator no longer deploys
  NetworkPolicies automatically (3.7.0). Supply policies separately when the
  cluster requires them.

## Upgrade verification checklist

1. Convert or replace Promtail before adopting a release where it is absent.
2. Search image customizations for `wget` and relative-path assumptions.
3. Replace the old object-store prefix key and render the chart.
4. Remove ksonnet-era configuration and audit deprecated stores and endpoints.
5. Compare structured metadata and stream-label collisions in representative
   entries.
6. Compare OpenShift stream labels and network isolation before and after
   Operator reconciliation.
7. Load-test scheduler behavior using the deployment's real worker and
   connection counts.
8. Update chart repositories and meta-monitoring ownership.
