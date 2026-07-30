---
name: elasticsearch-knowledge-patch
description: Elasticsearch
version: 9.4.0
license: MIT
metadata:
  author: Nevaberry
---


# Elasticsearch Knowledge Patch

Use this skill when designing, upgrading, configuring, querying, or operating
Elasticsearch and the work may depend on recent API, ES|QL, vector-search,
inference, data-stream, lifecycle, mapping, security, or storage behavior.

## How to use this skill

1. Determine the exact Elasticsearch version and deployment type.
2. Read the compatibility guidance before changing a cluster, client, plugin,
   index template, mapping, query, or lifecycle policy.
3. Open only the topic references needed for the task.
4. Treat technical-preview features as unstable interfaces and verify their
   availability in the target distribution.
5. Prefer the project's mappings, templates, settings, tests, and observed API
   responses when they differ from generic guidance.
6. For mixed-version clusters, use behavior supported by every participating
   node until the upgrade is complete.

## Reference index

| Reference | Topics |
| --- | --- |
| [compatibility-and-known-issues.md](references/compatibility-and-known-issues.md) | Breaking changes, removals, deprecations, upgrade hazards, and version-specific workarounds |
| [esql-and-querying.md](references/esql-and-querying.md) | ES|QL commands, joins, functions, time-series queries, partial results, views, PromQL, and cross-cluster/project search |
| [search-vectors-and-inference.md](references/search-vectors-and-inference.md) | `semantic_text`, dense and sparse vectors, DiskBBQ, retrievers, reranking, and Inference API changes |
| [data-streams-lifecycle-and-ingest.md](references/data-streams-lifecycle-and-ingest.md) | Failure stores, streams, ILM, downsampling, reindexing, transforms, and ingest processors |
| [mappings-time-series-and-observability.md](references/mappings-time-series-and-observability.md) | Mapping types and limits, synthetic source, TSDB identity, metrics, logs, and diagnostics |
| [security-cluster-and-storage.md](references/security-cluster-and-storage.md) | Authentication, TLS, entitlements, allocation, snapshots, repositories, connectors, and cluster operations |

## Breaking changes first

### Require complete ES|QL and EQL results explicitly

ES|QL may return partial results by default. Always inspect `is_partial` when
partial results are acceptable. For correctness-sensitive requests, send
`allow_partial_results=false`; the cluster-wide ES|QL setting is
`esql.query.allow_partial_results: false`.

EQL also defaults `allow_partial_search_results` to `true`. Opt out when every
shard must succeed.

When a remote cluster has `skip_unavailable: true`, ES|QL treats all of its
runtime errors, including missing indices, as non-fatal and reports that
cluster as skipped or partial.

### Account for source-vector exclusion

New indices enable `exclude_source_vectors` by default. Do not assume vector
values are returned from `_source`; request vector inclusion or use the Fields
API where supported. Reindex still carries vectors even when transparent
source removal hides them.

### Update ILM downsampling assumptions

ILM downsampling no longer force-merges by default. If a policy relies on a
merged downsampled index, add a force-merge action or set
`force_merge_index: true` on the downsample action.

### Update EC2 discovery configuration

The `discovery-ec2` plugin uses AWS SDK v2, requires IMDSv2, ignores
`discovery.ec2.protocol`, and removes legacy AWS SDK properties. Put the
scheme directly in `discovery.ec2.endpoint`. Configure both the access key and
secret key, or configure neither.

### Audit removed APIs, settings, and parameters

- Frozen indices and the unfreeze endpoint are removed.
- The technical-preview `_knn_search` API is removed; use supported kNN query
  and retriever forms.
- Highlighting no longer accepts `force_source`.
- Alias APIs no longer accept `local`.
- `/_cluster/reroute` responses no longer contain cluster state.
- The single-data-node disk-watermark setting is removed.
- `client.type`, `tracing.apm.*`, and
  `xpack.searchable.snapshot.allocate_on_rolling_restart` are removed.
- The `user_agent` processor no longer accepts `ecs`.
- Metadata-field definitions reject `type`, `fields`, `copy_to`, and `boost`.
- Watcher searches no longer accept `types`.

### Adjust changed defaults and parsing

- Timeouts return HTTP 429 rather than a 5xx response.
- Byte-size values accept at most two decimal places.
- `random_score` without a field uses `_seq_no`.
- `date_histogram` rejects boolean values.
- LogsDB and TSDB text fields omit norms.
- Logs data streams may use LogsDB automatically when enabling conditions hold.
- JDK 24 removes `TLS_RSA` ciphers and TLSv1.1 from defaults.
- An LDAP or Active Directory bind DN without a bind password prevents startup.
- ES|QL index-pattern quoting is all-or-nothing: quote the complete
  `remote:index` expression or none of it.

See the compatibility reference before any upgrade; it contains additional
analyzer, platform, connector-privilege, synthetic-source, OTLP, inference,
and Fleet-search changes.

## Deprecations to remove from new work

- Do not build new operational dependencies on ES|QL query logging.
- Avoid the deprecated `logs` data-stream type.
- Omit `aggregate_metric_double.default_metric` in new mappings.
- Replace ILM rollover `max_size` with supported conditions.
- Supply strict `true` or `false` values in settings and system properties.
- Remove `indices.merge.scheduler.use_thread_pool`.
- Write ES|QL metadata fields without brackets:

```esql
FROM my-index METADATA _id, _index
```

- Migrate away from the machine-learning flush API, the `elser` inference
  service, and Behavioral Analytics CRUD APIs.
- Recheck authorization assumptions around the built-in `reporting_user` role.

## High-value feature choices

### Choose a vector index deliberately

New vector indices and new `semantic_text` fields can default to DiskBBQ and
BFloat16 storage. DiskBBQ reduces memory pressure, supports configurable
quantization, and can keep raw rescoring vectors on disk, but its latency and
license constraints differ from HNSW. Use `flat_index_threshold` for small
HNSW fields and tune DiskBBQ kNN with `num_candidates` or
`visit_percentage`.

Use `rank_vectors` for late-interaction reranking when indexing every token
vector into HNSW is too costly. Use retrievers or ES|QL reranking when the
workflow needs fusion, normalization, contextual chunk scoring, or MMR result
diversification.

### Use `semantic_text` as an integrated workflow

`semantic_text` supports integrated inference, query, highlighting, chunks,
multi-fields, configurable vector index options, chunking, inference-ID
updates, and dense or sparse semantics. Empty content skips embedding
generation. Check mapping defaults during upgrades because the default
service, model, vector element type, and index type have changed.

### Plan failure stores as part of ingestion

Failure stores can capture documents rejected by pipelines or mappings.
Enable them in data-stream options or templates, query them with the
`::failures` selector, set lifecycle retention, and use the
`recover_failure_document` processor for remediation. New logs, telemetry, and
APM streams may enable them by default.

### Use ES|QL features by stability level

ES|QL provides joins, branching, vector search, inference, time-series
analytics, external sources, views, and PromQL integration. Some commands and
functions remain technical preview. Confirm stability before exposing syntax
through a public API or a durable saved query.

## Common workflows

### Upgrade a cluster

1. Identify every intermediate version and plugin.
2. Read all breaking changes, deprecations, and known issues that touch the
   path.
3. Test S3 and EC2 plugins because both moved to AWS SDK v2 with behavior and
   configuration differences.
4. Verify vector-source retrieval, partial-result handling, TLS configuration,
   analyzer output, and ILM downsampling.
5. Run repository analysis and representative search, ingest, inference,
   lifecycle, security, and recovery tests.

### Design an ES|QL endpoint

1. Decide whether partial results are allowed.
2. Validate index modes, remote-cluster behavior, and full-pattern quoting.
3. Request execution metadata when callers need shard or cluster detail.
4. Reject profiling with text response formats.
5. Preserve technical-preview status in the API contract.

### Design semantic or hybrid search

1. Select `semantic_text`, `dense_vector`, `sparse_vector`, or `rank_vectors`
   from the retrieval and storage requirements.
2. Choose HNSW, flat quantization, or DiskBBQ from memory, latency, dimension,
   precision, and licensing constraints.
3. Select a retriever or ES|QL pipeline for fusion, normalization, reranking,
   chunk scoring, and diversification.
4. Verify whether vectors are available through `_source`, the Fields API, or
   another explicit retrieval path.
5. Test defaults with the exact inference endpoint and model.

### Operate data streams

1. Inspect stream type, index mode, failure-store options, retention, and
   lifecycle state.
2. Query failures explicitly and preserve failure metadata during recovery.
3. Check child-stream indexing restrictions before enabling streams.
4. Verify downsampling method, force-merge behavior, counter-reset semantics,
   and follower ordering.
5. Treat TSDB synthetic IDs and disabled sequence numbers as constraints on
   ID-dependent operations.

## Validation principles

- Validate syntax against the target cluster rather than inferring support
  from a nearby release.
- Treat defaults as part of the versioned contract.
- Inspect warning headers and deprecation logs in automated tests.
- Exercise both success and partial/failure responses for distributed queries.
- Benchmark vector changes with representative dimensions, candidate counts,
  memory pressure, and rescoring.
- Test repository and security changes in a non-production cluster before
  rollout.
