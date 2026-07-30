---
name: grafana-loki-knowledge-patch
description: Grafana Loki
version: 3.7.0
license: MIT
metadata:
  author: Nevaberry
---



# Grafana Loki Knowledge Patch

Use this skill when upgrading, configuring, deploying, or debugging Grafana
Loki installations whose behavior may depend on recent Loki, Loki Operator,
Helm chart, LogQL, storage, or ingestion changes.

Check the deployed Loki, chart, and Operator versions before applying
version-sensitive guidance. Treat manifests, generated configuration, runtime
behavior, and focused tests as authoritative for the installation.

## How to use this skill

1. For an upgrade, start with the breaking-change checklist below and then
   open the migration reference.
2. Identify whether the deployment uses the Loki Helm chart, Loki Operator,
   SingleBinary, distributed mode, or a legacy deployment mode.
3. Open the task-specific reference from the index; several changes span both
   Loki configuration and deployment templates.
4. Verify tenant overrides separately from global defaults, especially for
   ingestion, limits, Kafka, and deletion processing.
5. Exercise storage and query changes against the configured backend before a
   production rollout.

## Reference index

| Reference | Topics |
| --- | --- |
| [migrations-and-breaking-changes.md](references/migrations-and-breaking-changes.md) | Promtail migration, removed tooling and ksonnet, deprecated stores and deployment modes, label precedence, scheduler behavior, and container assumptions |
| [helm-and-deployment.md](references/helm-and-deployment.md) | Chart ownership, templating, services, workloads, caches, persistence, probes, storage rendering, repository transfer, and IPv6 deployment behavior |
| [ingestion-labels-and-limits.md](references/ingestion-labels-and-limits.md) | Time sharding, structured metadata, relabeling, distributor limits, ingestion policies, Kafka, detected fields, and request handling |
| [queries-apis-and-cli.md](references/queries-apis-and-cli.md) | Query correctness, Parquet, endpoint routing, caches, Patterns and Drilldown APIs, limits, `logcli`, `lokitool`, health, and rule checking |
| [storage-deletion-and-compaction.md](references/storage-deletion-and-compaction.md) | Thanos clients, object stores, SQLite delete requests, scalable deletion workers, deletion markers, index-gateway sharding, and S3 compatibility |
| [operator-integrations-and-observability.md](references/operator-integrations-and-observability.md) | Loki Operator behavior, OpenShift, OTLP, tracing, Fluent plugins, meta-monitoring, and Operational UI integration |

## Breaking changes and deprecations

### Migrate away from Promtail

- Promtail was deprecated after its code moved into Grafana Alloy and was
  removed as of Loki 3.7.3.
- Use the Alloy migration documentation and configuration-conversion utility
  for Promtail configurations.
- Do not apply the Promtail removal to Lambda-promtail; it remains separate.

### Audit legacy storage and deployment choices

- BoltDB storage, legacy configuration options, and legacy API endpoints are
  deprecated. Inventory them before an upgrade.
- Deprecated ksonnet configuration was removed in 3.5.0.
- Simple Scalable Deployment is deprecated and scheduled for removal before
  Loki 4.0. The community `LGTM-distributed`, `loki-canary`,
  `loki-distributed`, and `loki-simple-scalable` charts are deprecated too.

### Review behavior changes that affect existing automation

- The Promtail image no longer includes `wget`; replace image probes, derived
  image steps, or scripts that invoke it.
- Operator OTLP attribute dropping was introduced as a breaking change.
- Parsed labels no longer override same-named structured metadata.
- Scheduler workers are shared across connections and scheduling accounts for
  total compute capacity; both engine changes are breaking.
- Loki images now use `/` as the working directory. Make relative paths in
  derived images and scripts explicit.
- Default OpenShift stream labels changed; review selectors, dashboards, and
  retention rules when upgrading the Operator.

## Per-tenant ingestion quick reference

### Time-shard long out-of-order streams

Enable time sharding in the applicable tenant override:

```yaml
shard_streams:
  time_sharding_enabled: true
```

Loki adds `__time_shard__` and limits each resulting stream to at most half of
`max_chunk_age`, normally one hour. This prevents very old entries in a long
stream from being rejected as too far behind. The Operator can also enable the
feature.

### Extract structured metadata at ingestion

Tenant configuration can extract fields from:

- ordinary stream labels;
- existing structured-metadata keys; and
- fields parsed from JSON or `logfmt` log lines.

Account for structured-metadata bytes in distributor sizing and limits. Loki
unescapes JSON metadata strings and suppresses duplicates that appear in both
stream labels and extracted fields.

### Apply limits in the right layer

- Distributors can enforce limits or evaluate them in dry-run mode.
- Aggregated metric streams bypass ordinary label enforcement.
- Per-policy stream limits override the applicable ingestion policy; default
  policy mappings are merged with tenant mappings rather than replaced.
- `MaxRecvMsgSize` controls the distributor's uncompressed message-size
  ceiling, and truncated lines receive an identifier.

## Query and API quick reference

### Protect correctness during upgrades

- Offsets now apply correctly to `last_over_time`, `first_over_time`, and
  `quantile_over_time`; `approx_topk` is mapped in all cases.
- The query-path `json` parser no longer risks corrupting log lines.
- In 3.7.3, range-query evaluation timestamps align to the step grid, and the
  engine no longer silently drops `OR` operations.
- Parsed labels yield to same-named structured metadata.

### Use current response and cache controls

- Query responses can use Parquet for direct columnar processing.
- Starting in 3.7.3, `query_range` requests can disable caching.
- The Patterns API accepts multi-tenant queries.
- Empty push requests return HTTP 422; interval-limit violations return HTTP
  400.

## Storage and deletion quick reference

### Check object-store client compatibility

- Object-store access moved to the shared Thanos client, including Swift
  support through `thanos.io/objstore`.
- Thanos storage prefixes accept dashes.
- S3 preserves regions already supplied by the configuration chain and adds a
  SHA-256 checksum for Object Lock `PutObject` calls as of 3.7.2.
- If the legacy S3 client uses `chunk_delimiter`, include the 3.7.4 index-file
  fix in validation.

### Scale deletion processing deliberately

The experimental horizontally scalable compactor delegates queued deletion
work to workers, but index compaction and retention remain in the singleton
Compactor. Object-backed chunk-deletion markers remove the local-disk
dependency; deployments using the Thanos filesystem backend also need the
3.7.4 delete-request repair.

## Helm deployment quick reference

### Render user-supplied templates consistently

The chart applies `tpl` to read, write, and backend pods and to
`pattern_ingester`, `ingester_client`, `loki.operational_config`, and
`nameOverride`. Validate rendered names and configuration with the exact chart
values used by the deployment.

### Plan chart ownership and repository changes

- The installation manager, rather than templates, sets `managed-by`.
- Checksums for ConfigMaps and Secrets cover only `.data`.
- The open-source Loki chart moved to `grafana-community/helm-charts` on
  March 16, 2026; update source references and automation. The GEL chart is
  maintained separately.

### Validate workload lifecycle controls

- Distributor and read workloads support startup probes, and SingleBinary
  accepts `topologySpreadConstraints`.
- Canary readiness is configurable, and the pod security context uses
  `fsGroupChangePolicy: OnRootMismatch`.
- StatefulSet PVCs remain on scale-down but may be deleted with the
  StatefulSet. Confirm claim-template labels, access modes, and
  `volumeAttributesClassName` before changing persistence.

## Operator and observability quick reference

- The Operator supports managed GCP Workload Identity, virtual-host-style S3,
  Swift TLS CAs, optional NetworkPolicies, custom gateway certificates, and
  suppression of ingress.
- OpenTelemetry replaces OpenTracing internally while retaining
  `JAEGER_`-prefixed configuration and Jaeger-format export.
- Metrics authentication no longer depends on `kube-rbac-proxy` as of 3.7.3.
- For OCP 4.20, do not assume the Operator creates NetworkPolicies; add the
  desired policy explicitly.

## Verification checklist

- Render Helm manifests and inspect effective Loki configuration.
- Test ingestion with representative labels, structured metadata, empty
  batches, oversized messages, and out-of-order streams.
- Compare query results around step boundaries, offsets, `OR`, label
  collisions, cache controls, and HTTP error behavior.
- Exercise deletion requests through the configured object-store client and
  backend.
- Confirm Operator-generated identities, certificates, networking, storage,
  and authorization on the target Kubernetes or OpenShift version.
