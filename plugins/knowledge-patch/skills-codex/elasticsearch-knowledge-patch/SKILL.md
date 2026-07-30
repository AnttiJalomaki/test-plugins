---
name: elasticsearch-knowledge-patch
description: Elasticsearch
version: 9.4.0
license: MIT
metadata:
  author: Nevaberry
---


# Elasticsearch Knowledge Patch

Use this skill when implementing, reviewing, upgrading, or operating modern
Elasticsearch clusters. Start with the compatibility checks below, then open
the topic reference that matches the work.

## Reference index

| Reference | Topics |
| --- | --- |
| [breaking-changes.md](references/breaking-changes.md) | Removed behavior, changed defaults, migration hazards |
| [deprecations-and-known-issues.md](references/deprecations-and-known-issues.md) | Deprecated APIs and settings, affected releases, fixes and workarounds |
| [esql.md](references/esql.md) | ES\|QL sources, joins, full text, vectors, time series, functions, result semantics |
| [search-and-vectors.md](references/search-and-vectors.md) | Retrievers, reranking, `semantic_text`, dense and sparse vectors |
| [inference.md](references/inference.md) | Inference API tasks, providers, endpoint controls, chunking |
| [data-mappings-and-ingest.md](references/data-mappings-and-ingest.md) | Data streams, failure stores, mappings, ingest, logs and metrics |
| [lifecycle-snapshots-and-storage.md](references/lifecycle-snapshots-and-storage.md) | ILM, downsampling, transforms, reindexing, snapshots, repositories |
| [security-cluster-and-operations.md](references/security-cluster-and-operations.md) | Security, federation, cluster APIs, runtime baselines, diagnostics |

## Apply the patch

1. Determine the exact Elasticsearch version and deployment type.
2. Check [breaking-changes.md](references/breaking-changes.md) before changing
   mappings, query behavior, plugins, TLS, ingest, or lifecycle policy.
3. Check
   [deprecations-and-known-issues.md](references/deprecations-and-known-issues.md)
   for the exact patch release, especially before an upgrade or rollback.
4. Use the task-specific reference for implementation details.
5. Prefer cluster mappings, settings, API responses, and tests over assumptions.
6. Treat technical-preview features as unstable contracts and gate them behind
   explicit deployment checks.
7. For APIs that can return partial results, make completeness an explicit
   application decision.

## Breaking-change quick reference

### Partial results

- ES|QL returns partial results by default. Inspect `is_partial`, or set
  `allow_partial_results=false` per request.
- EQL also defaults `allow_partial_search_results` to `true`.
- With a remote cluster configured as `skip_unavailable: true`, runtime errors
  such as missing indices are reported as skipped or partial, not fatal.
- Async result retrieval can expose intermediate results; callers must decide
  whether those are acceptable.

### Source and vector defaults

- New indices enable `exclude_source_vectors`, so vectors are not returned in
  `_source` unless requested through supported vector retrieval paths.
- New vector mappings default to DiskBBQ, and new `semantic_text` fields use
  DiskBBQ with BFloat16 storage.
- Reindex still carries vectors even when normal `_source` retrieval omits
  them.
- LogsDB and TSDB text fields omit norms.

### Lifecycle and downsampling

- ILM downsampling no longer force-merges by default. Add a force-merge action
  or set `force_merge_index: true` when policy behavior depends on it.
- The downsampling `aggregate` method preserves counter-reset information;
  `last_value` remains storage-oriented.
- Persistent-task reassignment during node shutdown is opt-in.

### Ingest and mapping validation

- Invalid ingest simulation requests return HTTP 400.
- Metadata field definitions reject `type`, `fields`, `copy_to`, and `boost`.
- `_source.mode` in the meta-field definition has no effect.
- `date_histogram` rejects boolean input.
- Byte-size values accept no more than two decimal places.
- `ignore_malformed` date fields do not silently accept objects or arrays.

### Security, TLS, and plugins

- Secure settings belong in the Elasticsearch secure-settings mechanism, not
  YAML.
- A configured LDAP or Active Directory bind DN must have a bind password or
  node startup fails.
- Connector APIs require `manage_connector` or `monitor_connector`.
- JDK 24 removes `TLS_RSA` cipher support, and TLSv1.1 is absent from defaults.
- `discovery-ec2` uses AWS SDK v2, requires IMDSv2, and has revised endpoint and
  credential configuration.
- Inference requests cannot override endpoint `secret_parameters` in affected
  patch releases.

### Removed interfaces

- The technical-preview `_knn_search` API, frozen-index reads, and the unfreeze
  endpoint are removed.
- Alias APIs no longer accept `local`; highlighting no longer accepts
  `force_source`.
- Cluster reroute responses no longer include cluster state.
- The old `data_frame_transforms` roles and Watcher search `types` field are
  removed.
- Fleet search endpoints operate only on the local cluster.

## Deprecation quick reference

- Do not build new operational dependencies on the ES|QL query log.
- Avoid the `logs` data-stream type in new definitions.
- Omit `aggregate_metric_double.default_metric` from new mappings.
- Replace ILM `max_size` rollover conditions.
- Use strict `true` and `false` values in plugin analysis settings and system
  properties.
- Remove `indices.merge.scheduler.use_thread_pool`.
- Write ES|QL `METADATA _id, _index` without brackets.
- Plan migrations away from the machine-learning flush API, the `elser`
  inference service, and Behavioral Analytics CRUD APIs.

## ES|QL design checklist

- Quote a whole remote index pattern or none of it; do not quote only one
  component.
- Check index-mode restrictions before using search functions.
- Treat `LOOKUP JOIN` capabilities as version-sensitive: aliases, multiple
  fields, expression predicates, remote input, and post-join operations arrived
  separately.
- When using `TS`, verify the request timestamp range, `TBUCKET` bounds, window
  size, reset-preserving downsampling, and series dimensions.
- Use `METRICS_INFO` for metric discovery and `TS_INFO` for metric-and-series
  discovery.
- Expect `FORK` to send every row through every branch and add `_fork`.
- Remember that `FORK` and subquery branches no longer inherit implicit limits.
- Profiling is invalid with text response formats.
- Views run their own pipelines and are unavailable when document- or
  field-level security applies.
- For vector queries, verify element type, quantization, candidate count,
  visit percentage, and whether raw-vector rescoring is in memory or on disk.

## Search and vector checklist

- Use `rank_vectors` for multi-vector late-interaction reranking where building
  HNSW for every vector would be too expensive.
- Select a retriever based on intent: RRF for rank fusion, linear for weighted
  normalized scores, MMR for diversification, pinned for curated placement,
  and rescorer retrievers for a second stage.
- Verify license requirements before creating DiskBBQ indices.
- Avoid DiskBBQ for low-dimensional vectors and remember that it accepts
  floating-point vectors.
- Use `on_disk_rescore` when raw vectors exceed available RAM.
- Use BFloat16 only when halved storage is worth its reduced precision.
- Treat direct I/O as workload-dependent; it can help under page-cache pressure
  but hurt when vectors fit in memory.
- Empty `semantic_text` content does not generate embeddings.
- A `semantic_text` field's `inference_id` can be updated, but defaults and
  provider availability vary by release.

## Inference checklist

- Distinguish endpoint creation settings, task settings, request settings, and
  secrets; do not assume they are interchangeable.
- Do not override stored secret parameters at request time.
- Validate task compatibility before force-deleting an invalid endpoint.
- Set query and transport timeouts deliberately; inference timeout responses
  use HTTP 504.
- Choose chunking explicitly when defaults matter; `none` disables automatic
  chunking and recursive chunking is available.
- Use data-URI form for base64 embedding inputs.
- Do not send `max_tokens` with reasoning chat requests.
- For provider migrations, verify API version, authentication method, aliases,
  rate limiting, license, and request field names.

## Data and ingest checklist

- Enable a failure store explicitly for existing streams; selected new logs,
  OTel, and APM streams enable one by default.
- Query rejected documents with the `::failures` selector.
- A logs document successfully redirected to a failure store returns HTTP 201
  and `"failure_store": "used"`.
- After enabling streams, index only to child streams and ensure conflicting
  indices do not exist.
- Use `recover_failure_document` when remediating failure-store records.
- Inspect the effective mapping returned by ingest simulation.
- For synthetic source, verify recovery-source behavior and the storage of text
  multi-fields.
- For TSDB doc-values skippers, check
  `index.mapping.use_doc_values_skipper` and per-field overrides.

## Upgrade triage

- Never jump to a known-bad patch release when a fixed patch is available.
- A direct upgrade from 9.1.10 to 9.2.4 can fail; target 9.2.5 or later.
- Upgrade 9.3.6 to 9.3.7 for trained-model request-limit fixes.
- Upgrade GCS ADC users from 9.2.8 to 9.2.9 or from 9.3.3 to 9.3.4.
- Upgrade mixed-GPU clusters from 9.3.1 to 9.3.2.
- Upgrade shrunk TSDB or LogsDB users from 9.1.0/9.1.1 to 9.1.2.
- Check old 7.x template residue if Watcher fails after reaching 9.x.
- Match Windows path casing exactly when entitlements are involved.
- Read the full workaround, scope, and cleanup steps in
  [deprecations-and-known-issues.md](references/deprecations-and-known-issues.md).

## Operational verification

- Confirm response status and semantic completion separately; HTTP success can
  still carry partial or redirected outcomes.
- Inspect thread-pool utilization and queue latency when diagnosing indexing
  pressure.
- Use circuit-breaker and shard-capacity health data before changing capacity.
- For vector performance, compare off-heap usage, page-cache pressure, direct
  I/O, candidate counts, and early termination.
- For S3 repositories, validate conditional-write compatibility and timeouts
  against the actual object store.
- Reload secure settings and confirm the returned setting names and keystore
  modification time.
- Exercise TLS reloads against the files themselves, including CSI-style
  symlink swaps where applicable.
