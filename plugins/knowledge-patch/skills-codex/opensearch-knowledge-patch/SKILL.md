---
name: opensearch-knowledge-patch
description: OpenSearch
version: 3.7.0
license: MIT
metadata:
  author: Nevaberry
---


# OpenSearch Knowledge Patch

Use this skill when planning, implementing, reviewing, or troubleshooting
OpenSearch clusters, clients, plugins, Dashboards, search applications, query
languages, security, or upgrades.

## Start with project reality

1. Determine the deployed OpenSearch and Dashboards versions from manifests,
   image tags, package metadata, or cluster responses.
2. Load only the reference files relevant to the task.
3. Apply a version-attributed note only when the target version includes that
   behavior.
4. Prefer the repository's configuration, code, tests, and observed cluster
   behavior when they disagree with general guidance.
5. Treat experimental and disabled-by-default features as opt-in. Verify their
   feature flags and production-readiness state for the exact target version.
6. For mixed-version upgrades, distinguish behavior on old nodes, new nodes,
   newly created indexes, and existing indexes.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/upgrades-and-core.md](references/upgrades-and-core.md) | Breaking changes, migration checks, runtime and node roles, indexing limits, storage, analyzers, and plugin compatibility |
| [references/vector-hybrid-and-relevance.md](references/vector-hybrid-and-relevance.md) | k-NN engines, compression, vector builds, hybrid and semantic search, star-tree, Learning to Rank, and relevance evaluation |
| [references/agents-ml-and-flows.md](references/agents-ml-and-flows.md) | ML Commons connectors, inference, agents, memory, tools, protocols, Flow Framework, and Launchpad |
| [references/ppl-sql-and-transport.md](references/ppl-sql-and-transport.md) | PPL and SQL syntax and semantics, Calcite behavior, query APIs, gRPC, Arrow, and pull-based ingestion |
| [references/security-and-multitenancy.md](references/security-and-multitenancy.md) | Authentication, authorization, API keys, certificates, security configuration, resource sharing, tenants, and remote metadata |
| [references/operations-observability-and-alerting.md](references/operations-observability-and-alerting.md) | Query Insights, workload management, metrics, traces, anomaly detection, alerting, scheduling, ISM, replication, and Dashboards |

## Breaking-change quick reference

### Before a 3.0 upgrade

- Reindex every index created before 2.x, including system indexes; 3.0 cannot
  open them.
- Move searchable-snapshot shards to nodes with the `warm` role. The `search`
  role is no longer sufficient.
- Run on JDK 21 and account for Lucene 10.1.0.
- Verify 3.x artifacts with `release@opensearch.org`, not the key reserved for
  2.x artifacts.
- Remove `_bulk?batch_size=...`, keep `_id` values within 512 bytes, and
  reshape JSON that exceeds parser nesting or property-name limits.
- Remove `mmap.extensions`, the `transport-nio` plugin, and
  `compatibility.override_main_response_version`.
- Replace `PathHierarchy` with `path_hierarchy`.
- Replace query-group endpoints, fields, and settings with workload-group
  forms.
- Replace `CatIndexTool` with `ListIndexTool`.
- Review Blake2b hash fixtures and reindex Romanian text affected by analyzer
  normalization.
- Update SQL pagination to Point in Time and remove OpenDistro endpoints,
  legacy settings, and removed SQL operations.
- Adjust automation for corrected Nodes API indexing-buffer field formats.
- Remove dependencies on the Dashboards DataGrid and
  `discover:newExperience`.

Read [references/upgrades-and-core.md](references/upgrades-and-core.md) before
performing any migration.

### Earlier upgrade traps

- Audit Elasticsearch OSS 6.8 documents for more than 10,000 nested objects
  before migrating them into OpenSearch.
- Install both Notifications backend plugins for OpenSearch 2.0 Alerting
  actions, and add the Dashboards plugin when managing them in the UI.
- Expect corrected case-insensitive wildcard queries to return fewer matches
  from 2.5 onward.
- Pin the k-NN engine when engine-specific storage matters; Faiss became the
  implicit engine in 2.18 and normalizes `cosinesimil` vectors.
- Map nested `text_embedding` input paths directly from 2.19 onward rather
  than using `_ingest._value`.

## High-value search changes

### Vector search

- Treat `index.knn` as creation-time immutable.
- Do not set both a training model ID and `dimension`; the training index
  supplies the dimension.
- Set `rescore: false` explicitly when rescoring must remain disabled. New
  OnDisk 4x-compressed indexes changed their rescore default in 3.1.0.
- Do not expect a terminal remote vector-build failure to fall back to CPU.
- Account for Faiss 32x compression using 1-bit scalar quantization by default
  in 3.6.0.
- Expect `docvalue_fields` vector retrieval to return base64-encoded binary by
  default in 3.7.0, not a numeric array.
- Use request-level opt-in for nested semantic `inner_hits` highlighting.

### Hybrid search

- Use RRF when rank-based fusion is preferable to score normalization.
- Use `hybrid_score_explanation` and `verbose_pipeline` to inspect scoring and
  processor transformations.
- Tune pagination depth, normalization bounds, Z-score normalization, RRF
  weights, `collapse`, grouped `inner_hits`, and `min_score` as supported by
  the target version.
- Keep `hybrid` at an allowed query position. From 3.6.0 it is rejected inside
  compound queries such as `function_score`, `constant_score`, and
  `script_score`.
- Use the hybrid optimizer and Search Relevance Workbench to compare
  normalization, RRF, judgments, metrics, and search configurations.

Read
[references/vector-hybrid-and-relevance.md](references/vector-hybrid-and-relevance.md)
for engine, semantic-field, sparse-retrieval, star-tree, and workbench details.

## High-value query-language changes

- Assume Calcite is the default PPL path, but distinguish unsupported-command
  fallback from failed-query behavior.
- Expect PPL `join` to default to `max=1` when
  `plugins.ppl.syntax.legacy.preferred=false`.
- Treat PPL final structs as maps. Missing `JSON_EXTRACT` paths and
  double-overflow results return null.
- Expect SQL pagination to use Point in Time and cursor continuation to remain
  within the original authorized indexes.
- Use the unified query APIs for shared parsing and profiling, but keep the
  query-only V2 path free of DML and DDL.
- Use `_tasks/_cancel` for cancellable PPL work and `fetch_size` where
  incremental retrieval is required.

Read [references/ppl-sql-and-transport.md](references/ppl-sql-and-transport.md)
before changing parsers, queries, pagination, or transport clients.

## High-value agent and ML changes

- Prefer Streamable HTTP for MCP integrations; the ML Commons MCP server
  deprecated SSE transport while separate inference streaming still uses SSE.
- Treat agentic memory, session deletion, context hooks, and message-history
  limits as explicit lifecycle decisions.
- Use processor chains for deterministic pre/post transformations and tool or
  model invocation.
- Validate connector endpoint trust, private-IP protection, ReDoS protection,
  header substitutions, and custom action methods.
- Prefer the production-ready unified registration API and
  `conversational_v2` behavior when targeting 3.7.0.
- Verify V2 `inferenceConfig.model_parameters`; they are honored in 3.7.0.

Read [references/agents-ml-and-flows.md](references/agents-ml-and-flows.md) for
the complete agent, memory, connector, tool, and flow contracts.

## Security and tenancy checklist

- Review the OpenSSL-provider removal and allowlist terminology before a 3.x
  security upgrade.
- Include `cluster:monitor/shards` where `_cat/shards` access is required.
- Choose and test certificate, JWT, JWKS, client-certificate, SPIFFE, Basic
  gRPC, and API-key authentication deliberately.
- Treat `plugins.security.dls.write_blocked` as a cluster-wide write guard for
  requests subject to document-level restrictions.
- Review renamed resource-access and Notifications settings during upgrade.
- Test centralized sharing migrations with an owner and default access level.
- Account for multi-tenant plugin routes and operations that are disabled or
  return 501.

Read [references/security-and-multitenancy.md](references/security-and-multitenancy.md)
before changing authentication, resource sharing, or tenant isolation.

## Operations checklist

- Separate live-query diagnosis, historical top-N analysis, recommendations,
  and workload-group controls.
- Treat the 3.1.0 list-based Alerting finding payload as temporary; 3.2.0
  restored one-finding-at-a-time publication.
- Use ISM simulation before applying transition changes, and use analyzer
  resource reloads instead of unnecessary reindexing where supported.
- Review Alerting multi-tenancy restrictions before enabling it.
- Configure external EventBridge scheduling with the required two-role model.
- Distinguish experimental SLO and unified-alert views from stable operations.

Read
[references/operations-observability-and-alerting.md](references/operations-observability-and-alerting.md)
for settings, APIs, response fields, and operational constraints.

## Review discipline

When reviewing an OpenSearch change:

1. Identify the exact component: core, Dashboards, Security, k-NN, Neural
   Search, ML Commons, SQL/PPL, Alerting, Anomaly Detection, Query Insights,
   Flow Framework, or another plugin.
2. Check whether the behavior is experimental, disabled by default,
   production-ready, generally available, deprecated, or removed.
3. Check whether the behavior applies to existing indexes, only newly created
   indexes, or both.
4. Verify API paths, request fields, response shapes, and renamed settings
   against the target version.
5. Add upgrade tests for changed defaults, corrected validation, fallback
   removal, authorization scope, and output representation.
6. Preserve explicit configuration when a default change would otherwise
   alter scoring, storage, pagination, scheduling, or access control.
