# Deprecations and known issues

This reference separates planned migrations from release-specific defects.
Apply workarounds only to the named affected versions and remove temporary
settings after upgrading.

## Deprecated query, data, and lifecycle interfaces

### ES|QL query logging

Starting in 9.4.2, using the ES|QL query log emits a deprecation message. Do not
introduce new operational dependencies on this log.

### The `logs` data-stream type

Elasticsearch 9.4.0 deprecates the `logs` stream type. Avoid it in new
data-stream definitions and plan to migrate existing uses.

### `aggregate_metric_double.default_metric`

Elasticsearch 9.4.0 deprecates the `default_metric` mapping parameter for
`aggregate_metric_double`. Omit it in new mappings and prepare existing
mappings for removal.

### ILM `max_size`

Elasticsearch 9.3.0 emits a deprecation notice for the `max_size` rollover
condition. Use supported rollover conditions instead of aggregate index size.

### ES|QL bracketed `METADATA`

Elasticsearch 9.0 drops bracketed `METADATA` syntax. List fields directly:

```esql
FROM my-index METADATA _id, _index
```

## Deprecated settings, privileges, and APIs

### Lenient booleans

Elasticsearch 9.3.0 warns about lenient boolean values in analysis settings
from third-party plugins and in boolean system properties. Use strict `true`
or `false`.

### `reporting_user` privileges

In 9.0.6 and 9.1.3, the built-in `reporting_user` role changed to derive
authorization from reserved Kibana privileges. Recheck custom assumptions that
depended on its former privilege composition.

### Merge-scheduler thread-pool setting

`indices.merge.scheduler.use_thread_pool` is deprecated as of 9.0.3. Remove it
from configurations prepared for later releases.

### Machine-learning flush API

Elasticsearch 9.0 emits a deprecation warning when the machine-learning flush
API is called. Stop treating that endpoint as a durable workflow dependency.

### `elser` inference service

Elasticsearch 9.0 deprecates the `elser` service in the Inference API. Avoid
new endpoints using it and plan migration for existing endpoints.

### Behavioral Analytics CRUD APIs

Elasticsearch 9.0 deprecates the Behavioral Analytics create, read, update,
and delete APIs. Applications need a migration plan before their removal.

## Upgrade blockers and fixed defects

### Trained-model request limits in 9.3.6

Elasticsearch 9.3.6 can reject otherwise valid create-trained-model requests
because it applies overly restrictive limits to `description`, `tags`,
`prefix_strings.ingest_prefix`, `prefix_strings.search_prefix`,
`input.field_names`, `default_field_map`, and `metadata`. Upgrade to 9.3.7.

### Direct upgrade from 9.1.10 to 9.2.4

A direct upgrade can prevent Elasticsearch from booting because stored
node-shutdown metadata has a field that 9.2.4 cannot parse. Use 9.2.5 or later.

### Mixed-GPU clusters in 9.3.1

On a multi-node 9.3.1 cluster with nodes lacking a GPU, `_xpack/usage` cannot
collect their GPU stats and repeatedly logs `OutboundHandler` serialization
warnings. Other usage data and single-node clusters are unaffected. Upgrade to
9.3.2, or temporarily suppress the flood:

```http
PUT /_cluster/settings
{
  "persistent": {
    "logger.org.elasticsearch.transport.OutboundHandler": "ERROR"
  }
}
```

### Shrunk TSDB and LogsDB merges in 9.1.0 and 9.1.1

An optimized merge path can make merges fail after shrinking TSDB or LogsDB
indices. Upgrade to 9.1.2. Until then, omit the post-shrink force merge from
ILM, or put this property on every data node and perform a rolling restart:

```text
-Dorg.elasticsearch.index.codec.tsdb.es819.ES819TSDBDocValuesConsumer.enableOptimizedMerge=false
```

Remove the property after upgrading because it slows merges.

### Low disk and shard closure in 9.0.3

Insufficient merge space can prevent a shard from closing, leaving index
closure or relocation hanging. This release mitigates the issue by defaulting
`indices.merge.disk.check_interval` to `0s`; do not manually enable that
disk-space check on 9.0.3.

### Incorrect two-key ES|QL groups

From 8.16.0 until fixes in 8.17.9, 8.18.7, and 9.0.4,
`STATS ... BY keyword1, keyword2` can return incorrect results when there are
exactly two keyword grouping fields and the first has more than 65,000 distinct
values. Upgrade, put the lower-cardinality field first, or filter to reduce
cardinality before `STATS`.

## Repository and storage issues

### GCS ADC failures in 9.2.8 and 9.3.3

`repository-gcs` operations using Application Default Credentials can fail
because an entitlement exception escapes credential-path discovery. Upgrade
9.2.8 to 9.2.9, or 9.3.3 to 9.3.4. If an immediate upgrade is impossible,
create `${ES_CONF_PATH}/jvm_options/workaround-gcsadc.options` with the policy
matching the installed version:

```text
# 9.2.8
-Des.entitlements.policy.repository-gcs=dmVyc2lvbnM6CiAgLSA5LjIuOApwb2xpY3k6CiAgQUxMLVVOTkFNRUQ6CiAgICAtIHNldF9odHRwc19jb25uZWN0aW9uX3Byb3BlcnRpZXMKICAgIC0gb3V0Ym91bmRfbmV0d29yawogICAgLSBmaWxlczoKICAgICAgICAtIHJlbGF0aXZlX3BhdGg6ICIuY29uZmlnL2djbG91ZCIKICAgICAgICAgIHJlbGF0aXZlX3RvOiBob21lCiAgICAgICAgICBtb2RlOiByZWFkCg==
# 9.3.3
-Des.entitlements.policy.repository-gcs=dmVyc2lvbnM6CiAgLSA5LjMuMwpwb2xpY3k6CiAgQUxMLVVOTkFNRUQ6CiAgICAtIHNldF9odHRwc19jb25uZWN0aW9uX3Byb3BlcnRpZXMKICAgIC0gb3V0Ym91bmRfbmV0d29yawogICAgLSBmaWxlczoKICAgICAgICAtIHJlbGF0aXZlX3BhdGg6ICIuY29uZmlnL2djbG91ZCIKICAgICAgICAgIHJlbGF0aXZlX3RvOiBob21lCiAgICAgICAgICBtb2RlOiByZWFkCg==
```

### DiskBBQ licensing after 9.2

Elasticsearch 9.2.0 failed to enforce the Enterprise license requirement for
`bbq_disk` indices. After upgrading to 9.3 or later, existing indices created
on 9.2 remain queryable and updatable, but creating new indices of this type
requires an Enterprise license.

### Direct I/O and `bbq_hnsw` in 9.1.0

In 9.1.0, `vector.rescoring.directio` defaults to `true` and can increase kNN
latency by up to tenfold for `bbq_hnsw` when vectors fit in memory. Set it to
`false` on every search node and restart:

```text
-Dvector.rescoring.directio=false
```

Remove the override in 9.1.1. New 9.1 indices with dense vectors over 384
dimensions default to `bbq_hnsw`.

### S3 repository analysis before 9.3.0

Repository analysis can incorrectly fail S3 linearizable-register checks
because multipart-upload semantics do not always satisfy the assumed
guarantees. Run analysis on a one-node cluster with
`?register_operation_count=1`, or upgrade to 9.3.0 or later.

## Entitlement and upgrade residue

### Windows path casing in 9.0

Elasticsearch 9.0 entitlements treat paths as case-sensitive even on Windows.
Casing mismatches can fail startup or block file operations with
`NotEntitledException`. Match exact filesystem casing in command-line paths,
configuration, environment variables, and secure settings.

### Active Directory authentication in 9.0

The 9.0 `x-pack-core` entitlement policy blocks the LDAP library's outbound
connection. As a temporary workaround, create
`${ES_CONF_PATH}/jvm_options/workaround-127061.options` containing:

```text
-Des.entitlements.policy.x-pack-core=dmVyc2lvbnM6CiAgLSA4LjE4LjAKICAtIDkuMC4wCnBvbGljeToKICB1bmJvdW5kaWQubGRhcHNkazoKICAgIC0gc2V0X2h0dHBzX2Nvbm5lY3Rpb25fcHJvcGVydGllcwogICAgLSBvdXRib3VuZF9uZXR3b3Jr
```

### Watcher after an old 7.x upgrade

A 9.x cluster that previously ran any release from 7.10.0 through 7.12.1 can
retain templates that prevent Watcher from starting. Delete them and restart
Watcher:

```http
DELETE _index_template/.triggered_watches
DELETE _index_template/.watches
POST /_watcher/_start
```
