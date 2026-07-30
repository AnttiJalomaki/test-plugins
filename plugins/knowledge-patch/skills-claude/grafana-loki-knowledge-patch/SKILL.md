---
name: grafana-loki-knowledge-patch
description: Grafana Loki
version: 3.7.0
license: MIT
metadata:
  author: Nevaberry
---


# Grafana Loki

Load this skill when upgrading, configuring, deploying, operating, or integrating
Grafana Loki. Start with the upgrade hazards below, then open the topic reference
that matches the work.

## Reference index

| Reference | Topics |
| --- | --- |
| [upgrades-and-deprecations.md](references/upgrades-and-deprecations.md) | Breaking behavior, removals, deprecations, and migration checkpoints |
| [helm-and-operator.md](references/helm-and-operator.md) | Helm values and rendering, workloads, persistence, Loki Operator, and OpenShift |
| [ingestion-and-metadata.md](references/ingestion-and-metadata.md) | Push handling, sharding, limits, labels, structured metadata, OTLP, and agents |
| [queries-apis-and-patterns.md](references/queries-apis-and-patterns.md) | LogQL, query correctness, caching, APIs, patterns, limits, and scheduler behavior |
| [storage-deletion-and-kafka.md](references/storage-deletion-and-kafka.md) | Object stores, TSDB, deletion, Kafka, compactor, index gateway, and caches |
| [clients-observability-and-operations.md](references/clients-observability-and-operations.md) | CLI tools, tracing, UI, monitoring, networking, commands, and containers |

## Upgrade hazards first

### Replace Promtail with Grafana Alloy

Promtail was deprecated after its code moved to Grafana Alloy, then removed in
3.7.3. Use Alloy's migration documentation and configuration-conversion
utility. Do not apply the removal to Lambda-promtail; it remains a separate
component.

The Promtail image also stopped shipping `wget`. Replace probes, scripts, or
derived-image steps that call it before adopting the affected image.

### Review storage and legacy configuration

BoltDB storage and additional legacy configuration and API endpoints are
deprecated. Audit them before an upgrade. Removed ksonnet configurations cannot
be carried forward.

For Helm object-store values, use:

```yaml
object_store:
  storage_prefix: loki
```

Do not retain the former `object_store.prefix` key.

### Recheck labels and metadata

Parsed labels no longer override structured metadata with the same name. This
is a breaking semantic change, so test parsers and queries that depend on
name collisions.

Operator-managed OTLP attribute dropping and the changed default OpenShift
stream labels are also breaking changes. Compare generated ingestion behavior
before rollout.

### Rebaseline scheduling

The scheduler accounts for total compute capacity and shares worker threads
across scheduler connections. Both changes can alter query execution and are
classified as breaking; load-test concurrency and capacity assumptions.

### Update deployment assumptions

Simple Scalable Deployment is deprecated and scheduled for removal before Loki
4.0. The community `LGTM-distributed`, `loki-canary`, `loki-distributed`, and
`loki-simple-scalable` charts are deprecated as well.

The open-source Loki chart moved to `grafana-community/helm-charts`. Update
source references and chart-update automation. The GEL chart is maintained
separately.

Loki container images now use `/` as their working directory. Make paths
explicit in derived images and scripts that previously relied on a relative
working directory.

## High-value ingestion controls

### Enable old-log time sharding per tenant

```yaml
shard_streams:
  time_sharding_enabled: true
```

Loki adds `__time_shard__` so a resulting stream covers at most half of
`max_chunk_age`, normally one hour. This allows long out-of-order ingestion
without rejecting very old entries as too far behind.

### Extract structured metadata during ingestion

Tenant configuration can promote values from ordinary labels, existing
structured-metadata keys, or fields parsed from JSON and `logfmt` lines into
structured metadata. Account for metadata bytes in limit planning, and avoid
duplicating a value that is already supplied as a stream label.

### Apply limits at the distributor

Distributor limit checks can enforce or dry-run. Aggregated metric streams are
exempt from normal label enforcement, and rate-limit reasons identify stream
labels. `MaxRecvMsgSize` controls the uncompressed request limit.

Ingestion policies can override stream limits. Default policy mappings merge
with tenant mappings; a tenant mapping does not replace the defaults wholesale.

### Use tenant-specific Kafka ingestion

Kafka ingestion can select topics per tenant, consume records through multiple
clients, and feed the block-building path. Use the Helm `block_builder`
configuration when deploying that path.

## High-value query and API controls

### Return Parquet

Select Parquet from the query API when downstream processing benefits from a
columnar response rather than decoding the ordinary query response.

### Disable range-query caching when necessary

From 3.7.3, a `query_range` request can opt out of caching. Use this for request
paths where a cached result is undesirable; do not disable caching globally
without measuring the cost.

### Scale deletion processing

The experimental horizontally scalable compactor delegates queued deletion
work to workers. This scales large deletes and backlogs, while index compaction
and retention stay in the singleton Compactor.

Deletion requests can also use SQLite. Query-time filtering uses each request's
stored completion time to reduce the set it must consider.

### Persist detected patterns

Behind its feature flag, the pattern ingester can persist patterns as
aggregated metrics for later queries. Bound persistence by volume and
frequency; volume-based filtering is available, and detected level is emitted
as structured metadata. The Patterns API accepts multi-tenant queries.

## High-value Helm controls

### Treat selected values as templates

The chart evaluates `tpl` for read, write, and backend pods and for
`pattern_ingester`, `ingester_client`, `loki.operational_config`, and
`nameOverride`. Preserve deliberate literal template syntax with the
appropriate Helm escaping.

### Choose chart-managed or full storage configuration

The chart exposes full storage configuration, can bypass its generated
S3/GCS/Azure configuration, and permits separate ruler storage. Avoid mixing
the generated and bypass paths accidentally.

### Tune workload placement and persistence

Use topology spread constraints for admin API, distributed, and SingleBinary
workloads where supported. Configure PVC access modes, claim-template labels,
and `volumeAttributesClassName` deliberately. StatefulSet scale-down retains
PVCs, while StatefulSet deletion may delete them.

### Control the Operator's network behavior

The Operator can create NetworkPolicies with a LokiStack, suppress ingress,
customize the gateway server certificate, configure Swift TLS CA material, and
use virtual-host-style S3 access. On OpenShift 4.20 it no longer creates
NetworkPolicies automatically.

## High-value operational changes

### Keep Jaeger environment configuration during tracing migration

Loki uses OpenTelemetry internally instead of OpenTracing, but still accepts
`JAEGER_`-prefixed environment configuration and exports Jaeger-format traces.

### Inspect effective tenant limits

Use the applied-limits endpoint to retrieve a tenant's configured limits and
optionally filter fields with an allowlist. A nonexistent tenant returns
default limits.

### Use the current operational commands

`logcli` includes deletion commands and custom request headers. `loki health`
provides a health command, and the ruler checker can validate a namespace and
group. `lokitool` supports regex namespace filtering and the updated ruler
path.

## Working method

1. Identify whether the task concerns an upgrade, Helm, the Operator,
   ingestion, queries, storage, or operations.
2. Read the matching reference before changing configuration.
3. Check the breaking and deprecated behavior for any adjacent subsystem.
4. Preserve tenant-specific semantics when applying global defaults.
5. Validate rendered Helm resources and generated Operator resources, not only
   their input values.
6. Exercise ingestion, queries, deletion, and storage against the actual
   backend in a staging environment.
7. Confirm HTTP status codes and response formats in clients that make strict
   assumptions.
