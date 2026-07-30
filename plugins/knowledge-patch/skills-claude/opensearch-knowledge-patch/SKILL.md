---
name: opensearch-knowledge-patch
description: OpenSearch
version: 3.7.0
license: MIT
metadata:
  author: Nevaberry
---


# OpenSearch Knowledge Patch

Use this skill when designing, upgrading, operating, or debugging OpenSearch
clusters and OpenSearch Dashboards. Start with the upgrade checks below, then
load the topic reference that matches the task. Treat cluster mappings,
settings, installed plugins, security configuration, and observed API behavior
as authoritative when they differ from assumptions.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/upgrades-and-platform.md](references/upgrades-and-platform.md) | Breaking changes, migration blockers, runtime and packaging changes, removed settings |
| [references/vector-and-neural-search.md](references/vector-and-neural-search.md) | k-NN, Faiss, Lucene vectors, compression, semantic fields, sparse retrieval |
| [references/search-relevance-and-insights.md](references/search-relevance-and-insights.md) | Hybrid search, scoring, aggregations, relevance evaluation, Query Insights, workloads |
| [references/ppl-and-sql.md](references/ppl-and-sql.md) | Calcite, PPL commands and semantics, SQL, unified query APIs, PPL monitors |
| [references/agents-ml-and-flows.md](references/agents-ml-and-flows.md) | ML Commons, agents, memory, tools, MCP, connectors, Flow Framework |
| [references/security-and-multitenancy.md](references/security-and-multitenancy.md) | Authentication, authorization, certificates, API keys, resource sharing, tenancy |
| [references/cluster-operations-and-lifecycle.md](references/cluster-operations-and-lifecycle.md) | Remote store, ingestion, gRPC, ISM, replication, scheduler, metadata |
| [references/observability-alerting-and-dashboards.md](references/observability-alerting-and-dashboards.md) | Metrics, traces, Discover, Alerting, Anomaly Detection, notifications |

## Upgrade checks first

### Block unsafe major upgrades

- Reindex every index created before 2.x, including system indexes, before a
  3.0 upgrade.
- Audit documents migrated from Elasticsearch OSS 6.8 for more than 10,000
  nested objects; such documents can block shard relocation.
- Move searchable-snapshot shards to nodes with the `warm` role. The `search`
  role is no longer valid for those shards.
- Reshape JSON deeper than 1,000 object or array levels and property names
  beyond the 50,000-byte-or-character parser limit.
- Install the backend notification plugins required by Alerting before a 2.0
  migration, and install the Dashboards notification plugin for UI management.

### Update removed and renamed interfaces

- Replace `compatibility.override_main_response_version`; it no longer changes
  the root response.
- Replace `wlm/query_group` with `wlm/workload_group`,
  `queryGroupID` with `workloadGroupID`, and the old cluster-setting prefix
  with `wlm.workload_group`.
- Replace `CatIndexTool` with `ListIndexTool`.
- Replace `PathHierarchy` with `path_hierarchy` in analyzer configuration.
- Replace whitelist-named Security settings with their allowlist equivalents.
- Remove `_bulk?batch_size=...`, `mmap.extensions`, the `transport-nio` plugin,
  old index-level k-NN tuning settings, and legacy OpenDistro SQL interfaces.
- Review SQL clients for Point-in-Time pagination and corrected Nodes API
  indexing-buffer field formats.

### Verify runtime and cryptographic assumptions

- Run OpenSearch 3.x on JDK 21 or newer.
- Verify 3.x artifacts with the `release@opensearch.org` signing key.
- Retest Blake2b hash fixtures because corrected salt handling changes output.
- Reindex Romanian text when mixing old and newly normalized analyzer output.
- Update Dashboards build and runtime integrations for Node.js 22 and Rspack.

## High-value search guidance

### Make k-NN choices explicit

- The implicit engine changed from NMSLIB to Faiss. Set `engine` explicitly
  when stored-vector representation or scoring stability matters.
- With implicit Faiss and `space_type: "cosinesimil"`, indexing normalizes
  vectors. Already-normalized vectors can use `innerproduct` for equivalent
  scoring without implicit normalization.
- A model-backed vector mapping must not specify both model ID and `dimension`.
  The training index supplies the dimension.
- `index.knn` is immutable and derived vector source is incompatible with
  `index.knn: false`.
- Remote vector building is enabled by default. A terminal remote-build
  failure does not fall back to CPU.
- New OnDisk indexes using 4x compression rescore by default. Set
  `rescore: false` to opt out.

### Account for compressed-vector behavior

- Faiss 32x compression defaults to scalar-quantized 1-bit encoding.
- Use asymmetric distance computation and random rotation where their recall
  tradeoffs fit binary-quantized indexes.
- `docvalue_fields` returns compressed `knn_vector` values as base64-encoded
  binary by default, not as numeric arrays.
- Retrieve the full vector and decode it deliberately before comparing it with
  the original input.

### Build valid hybrid queries

- Use RRF or score normalization deliberately; Z-score, min-max lower and upper
  bounds, custom RRF weights, and optimizer-selected `rank_constant` values
  have distinct ranking effects.
- Use `pagination_depth` for large hybrid result windows.
- Hybrid queries support collapse and group `inner_hits`, but invalid nested
  hybrid structures are rejected.
- Do not place a `hybrid` query inside `function_score`, `constant_score`,
  `script_score`, or another compound query.
- Use `hybrid_score_explanation` and `verbose_pipeline` to inspect ranking and
  processor transformations.

## High-value query-language guidance

### Expect Calcite semantics

- Calcite is the default PPL path.
- General Calcite query failures do not fall back to the v2 engine, while
  unsupported commands can route to v2.
- Date and time functions default to UTC across PPL and SQL.
- `query.size_limit` limits final results, not intermediate pipeline rows.
- `NOT IN` and `NOT LIKE` exclude missing values as well as explicit nulls.
- Calcite `dedup` preserves sort order, and wildcard searches across text and
  keyword mappings no longer silently discard documents.

### Protect pagination and access boundaries

- SQL pagination defaults to Point in Time; do not depend on the removed
  pagination setting or deprecated Scroll behavior.
- Under fine-grained access control, SQL cursor continuation stays within the
  indexes selected by the original query.
- The unified V2 query path is query-only and rejects DML and DDL.
- Cancel PPL through `_tasks/_cancel`, and use `fetch_size` where result
  pagination is required.

## High-value security guidance

### Validate authorization at every interface

- Account for the extra `cluster:monitor/shards` permission used by
  `_cat/shards`.
- Treat resource sharing, tenant visibility, DLS/FLS, workload filtering, and
  plugin multi-tenancy as separate enforcement layers.
- Use the permission-validation request mode before execution when checking a
  principal's access.
- For gRPC, configure TLS plus a supported Security authentication method;
  Basic and JWT authentication are available.
- Scope API-key permissions directly on each key, set expiration, and plan for
  synchronous cluster-wide revocation.

### Review dynamic configuration carefully

- Security cache TTL changes are picked up dynamically.
- Resource settings and preferred Dashboards tenants can also be updated
  without a restart.
- Static security configuration wins when it overlaps custom configuration.
- The notification resource-access filename and settings prefix changed; audit
  both during upgrade.

## High-value operations guidance

### Separate readers and writers deliberately

- On remote-store clusters, `_scale` can remove writers and make an index
  search-only.
- ISM also supplies `search_only`, policy simulation, transition exclusions,
  `no_alias`, and `min_state_age`.
- Policy simulation reads live metrics and reports the next state without
  changing cluster state.
- Replication lifecycle APIs can clear stale persistent tasks and accept
  `cluster_manager_timeout`.

### Observe before tuning

- Query Insights exposes live-query, historical top-N, profiling,
  recommendation, and remote-export paths.
- Workload groups can auto-tag requests and override timeouts, cancellation
  intervals, and bucket limits.
- Filter live-query and historical data by the same identity and backend-role
  rules used for non-admin access.
- Inspect shard-level live-query tasks and recently finished-query history
  before changing cancellation or workload thresholds.

## Feature-state discipline

- Disabled-by-default features must be enabled intentionally and tested with
  the installed plugin set.
- Do not assume an earlier experimental transport, agent, search, or
  observability feature is still experimental; several graduated later.
- Conversely, do not retain removed experimental PPL Alerting assets or legacy
  endpoint paths after their transition.
- When a feature changes state, use the latest applicable behavior and keep the
  earlier state only as upgrade context.

## Task workflow

1. Identify the installed OpenSearch and Dashboards versions and plugin set.
2. Read the relevant upgrade and topic references from the index.
3. Check whether the behavior is core, plugin-specific, experimental, or
   disabled by default.
4. Compare mappings, persistent and transient settings, security configuration,
   node roles, and request payloads with the documented constraints.
5. Reproduce against a non-production index or cluster when changing mappings,
   codecs, vector representation, authentication, or lifecycle policy.
6. Use explain, stats, profiling, live-query, simulation, or dry-run APIs where
   the subsystem provides them.
7. Roll out with explicit rollback criteria and verify stored data as well as
   response shape; several changes affect one but not the other.
