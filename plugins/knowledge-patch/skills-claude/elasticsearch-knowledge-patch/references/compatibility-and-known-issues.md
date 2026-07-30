# Compatibility, Deprecations, and Known Issues

Use this reference for upgrade planning and version-specific incident
response. Confirm the installed patch version before applying a workaround.

## Contents

- [Breaking query and response behavior](#breaking-query-and-response-behavior)
- [Removed interfaces and settings](#removed-interfaces-and-settings)
- [Changed platform and index defaults](#changed-platform-and-index-defaults)
- [Lifecycle and plugin breaks](#lifecycle-and-plugin-breaks)
- [Deprecations](#deprecations)
- [Known upgrade and runtime issues](#known-upgrade-and-runtime-issues)

## Breaking query and response behavior

- ES|QL partial results are enabled by default. Inspect `is_partial`, set
  `allow_partial_results=false` per request, or set
  `esql.query.allow_partial_results: false` cluster-wide.
- EQL similarly defaults `allow_partial_search_results` to `true`.
- With `skip_unavailable: true`, an ES|QL remote-cluster runtime error,
  including a missing index, becomes non-fatal and the cluster is reported as
  skipped or partial.
- ES|QL index patterns must quote the whole remote-and-index expression or no
  part of it. `FROM "remote:index"` and `FROM remote:index` are valid;
  `FROM remote:"index"` is invalid.
- Elasticsearch timeouts return HTTP 429 instead of a 5xx response, and byte
  sizes are limited to two decimal places.
- `date_histogram` no longer accepts boolean values.
- `random_score` without an explicit field uses `_seq_no`.
- `_fleet/_fleet_search` and `_fleet/_fleet_msearch` are local-only.
- Inference requests can no longer override `secret_parameters` in 9.3.8 and
  9.4.4.
- Invalid processors in simulate-ingest requests now produce HTTP 400.

## Removed interfaces and settings

- Highlighting rejects `force_source`; alias APIs reject `local`; frozen
  indices cannot be read; and the unfreeze REST endpoint is removed.
- The technical-preview `_knn_search` endpoint is removed.
- `/_cluster/reroute` responses no longer include cluster state, and
  `cluster.routing.allocation.disk.watermark.enable_for_single_data_node` is
  removed.
- Removed settings include `client.type`, `tracing.apm.*`, and
  `xpack.searchable.snapshot.allocate_on_rolling_restart`.
- The `user_agent` ingest processor no longer accepts `ecs`, and the GeoIP
  processor's ignored fallback option is removed.
- Metadata-field definitions no longer accept `type`, `fields`, `copy_to`, or
  `boost`; `_source.mode` is a no-op.
- Machine learning is disabled on macOS x86_64. The
  `data_frame_transforms` roles and Watcher search `types` field are removed.

## Changed platform and index defaults

- Elasticsearch 9.0.0 bundles JDK 24, uses Lucene 10.1.0, and changes the
  default container image from Ubuntu to UBI minimal. Startup ignores
  `_JAVA_OPTIONS`.
- Entitlements permanently replace the Java SecurityManager in 9.0.0.
- JDK 24 removes `TLS_RSA` cipher support and TLSv1.1 from the default
  protocols.
- New indices exclude vectors from `_source` by default.
- Normalized `keyword` fields use native synthetic source.
- LogsDB and TSDB text fields omit norms.
- LogsDB is conditionally enabled for `logs-*-*` data streams.
- The OTLP endpoint maps histograms as `exponential_histogram` by default.
- Analyzer output may change after Snowball and Nori dictionary updates.
  `german2` aliases the `german` Snowball stemmer, and the `persian` analyzer
  stems by default.
- A configured LDAP or Active Directory bind DN without its bind password
  prevents node startup.
- Connector APIs require `manage_connector` or `monitor_connector`.
- The deprecation-log keyword is `elasticsearch.deprecation`, replacing
  `deprecation.elasticsearch`.

## Lifecycle and plugin breaks

- Starting in 9.4.0, ILM downsampling does not force-merge by default. Add a
  force-merge action or set `force_merge_index: true` when required.
- `discovery-ec2` uses AWS SDK v2, requires IMDSv2, ignores
  `discovery.ec2.protocol`, and removes `aws.secretKey` and
  `com.amazonaws.sdk.ec2MetadataServiceEndpointOverride`. Include `http://`
  in `discovery.ec2.endpoint` when needed and set both access and secret keys
  or neither.

## Deprecations

- ES|QL query logging emits deprecation messages starting in 9.4.2. Do not
  create new dependencies on it.
- The `logs` data-stream type is deprecated in 9.4.0.
- `aggregate_metric_double.default_metric` is deprecated in 9.4.0; omit it
  from new mappings.
- ILM's `max_size` rollover condition is deprecated in 9.3.0. Move to
  supported rollover conditions.
- Lenient boolean analysis settings and boolean system properties are
  deprecated in 9.3.0; use strict `true` or `false`.
- The built-in `reporting_user` role derives authorization from reserved
  Kibana privileges in 9.0.6 and 9.1.3. Recheck custom assumptions based on
  its former privilege composition.
- `indices.merge.scheduler.use_thread_pool` is deprecated as of 9.0.3.
- ES|QL bracketed `METADATA` syntax is removed in 9.0.0. Write:

```esql
FROM my-index METADATA _id, _index
```

- The machine-learning flush API, the Inference API `elser` service, and
  Behavioral Analytics CRUD APIs are deprecated in 9.0.0.

## Known upgrade and runtime issues

### Trained-model request rejection

Elasticsearch 9.3.6 applies overly restrictive create-trained-model limits to
`description`, `tags`, both `prefix_strings` fields, `input.field_names`,
`default_field_map`, and `metadata`. Upgrade to 9.3.7.

### GCS Application Default Credentials

In 9.2.8 and 9.3.3, `repository-gcs` can fail while discovering ADC paths
because of an entitlement exception. Upgrade to 9.2.9 or 9.3.4. When an
immediate upgrade is impossible, create
`${ES_CONF_PATH}/jvm_options/workaround-gcsadc.options` with the matching
temporary policy:

```text
# 9.2.8
-Des.entitlements.policy.repository-gcs=dmVyc2lvbnM6CiAgLSA5LjIuOApwb2xpY3k6CiAgQUxMLVVOTkFNRUQ6CiAgICAtIHNldF9odHRwc19jb25uZWN0aW9uX3Byb3BlcnRpZXMKICAgIC0gb3V0Ym91bmRfbmV0d29yawogICAgLSBmaWxlczoKICAgICAgICAtIHJlbGF0aXZlX3BhdGg6ICIuY29uZmlnL2djbG91ZCIKICAgICAgICAgIHJlbGF0aXZlX3RvOiBob21lCiAgICAgICAgICBtb2RlOiByZWFkCg==
# 9.3.3
-Des.entitlements.policy.repository-gcs=dmVyc2lvbnM6CiAgLSA5LjMuMwpwb2xpY3k6CiAgQUxMLVVOTkFNRUQ6CiAgICAtIHNldF9odHRwc19jb25uZWN0aW9uX3Byb3BlcnRpZXMKICAgIC0gb3V0Ym91bmRfbmV0d29yawogICAgLSBmaWxlczoKICAgICAgICAtIHJlbGF0aXZlX3BhdGg6ICIuY29uZmlnL2djbG91ZCIKICAgICAgICAgIHJlbGF0aXZlX3RvOiBob21lCiAgICAgICAgICBtb2RlOiByZWFkCg==
```

### Mixed-GPU log flooding

On a multi-node 9.3.1 cluster where some nodes lack GPUs, `_xpack/usage` can
repeatedly log `OutboundHandler` serialization warnings while GPU stats fail.
Other usage data and single-node clusters are unaffected. Upgrade to 9.3.2,
or temporarily suppress the flood:

```http
PUT /_cluster/settings
{
  "persistent": {
    "logger.org.elasticsearch.transport.OutboundHandler": "ERROR"
  }
}
```

### Incompatible direct upgrade

A direct upgrade from 9.1.10 to 9.2.4 can fail at startup because node-shutdown
metadata contains a field 9.2.4 cannot parse. Upgrade to 9.2.5 or later.

### DiskBBQ licensing

Elasticsearch 9.2.0 did not enforce the Enterprise requirement for
`bbq_disk`. After upgrading to 9.3.0 or later, existing indices remain
queryable and updatable, but creating new ones requires an Enterprise license.

### Shrunk TSDB and LogsDB merges

In 9.1.0 and 9.1.1, an optimized merge path can fail after shrinking TSDB or
LogsDB indices. Upgrade to 9.1.2. Until then, omit the post-shrink ILM force
merge or put this property on every data node and perform a rolling restart;
remove it after upgrade because it slows merges:

```text
-Dorg.elasticsearch.index.codec.tsdb.es819.ES819TSDBDocValuesConsumer.enableOptimizedMerge=false
```

### Direct I/O and `bbq_hnsw`

In 9.1.0, `vector.rescoring.directio=true` can make kNN against `bbq_hnsw`
indices up to ten times slower when vectors fit in memory. Set it to `false`
on every search node and restart. Remove the override in 9.1.1. New 9.1
indices with dense vectors over 384 dimensions default to `bbq_hnsw`.

```text
-Dvector.rescoring.directio=false
```

### Low-disk shard closure

In 9.0.3, a merge without enough free space can prevent shard closure and hang
index closure or relocation. Keep the release default
`indices.merge.disk.check_interval: 0s`; do not enable that check manually.

### Incorrect two-key ES|QL grouping

From 8.16.0 until fixes in 8.17.9, 8.18.7, and 9.0.4,
`STATS ... BY keyword1, keyword2` can be wrong when exactly two keyword fields
are used and the first exceeds 65,000 distinct values. Upgrade, put the
lower-cardinality field first, or filter to lower cardinality.

### Windows entitlement paths

Elasticsearch 9.0.0 entitlements compare Windows paths case-sensitively. Match
filesystem casing exactly in command-line paths, configuration, environment
variables, and secure settings to avoid startup failures or
`NotEntitledException`.

### Active Directory connectivity

The 9.0.0 `x-pack-core` entitlement policy blocks the LDAP library's outbound
connection. As a temporary workaround, create
`${ES_CONF_PATH}/jvm_options/workaround-127061.options` containing:

```text
-Des.entitlements.policy.x-pack-core=dmVyc2lvbnM6CiAgLSA4LjE4LjAKICAtIDkuMC4wCnBvbGljeToKICB1bmJvdW5kaWQubGRhcHNkazoKICAgIC0gc2V0X2h0dHBwc19jb25uZWN0aW9uX3Byb3BlcnRpZXMKICAgIC0gb3V0Ym91bmRfbmV0d29yaw
```

### Watcher after an old 7.x upgrade

A cluster previously on 7.10.0 through 7.12.1 can retain templates that stop
Watcher in 9.x. Delete them and restart Watcher:

```http
DELETE _index_template/.triggered_watches
DELETE _index_template/.watches
POST /_watcher/_start
```

### S3 repository analysis

Before 9.3.0, repository analysis can incorrectly fail S3
linearizable-register checks because multipart uploads do not always meet the
assumed guarantees. Run analysis on a single-node cluster with
`?register_operation_count=1`, or upgrade.
