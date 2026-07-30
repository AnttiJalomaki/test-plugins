---
name: qdrant-knowledge-patch
description: Qdrant
version: 1.18.0
license: MIT
metadata:
  author: Nevaberry
---


# Qdrant Knowledge Patch

Use this skill when designing, implementing, upgrading, or operating Qdrant
collections and clusters. Start with the quick reference, then open the topic
reference that matches the work.

## Reference index

| Reference | Topics |
| --- | --- |
| [deployment-upgrades-and-operations.md](references/deployment-upgrades-and-operations.md) | Upgrade sequencing, rolling availability, Helm, Gridstore, write back pressure |
| [indexing-storage-and-quantization.md](references/indexing-storage-and-quantization.md) | GPU and incremental HNSW, quantization, inline vectors, per-field graph participation |
| [retrieval-ranking-and-filtering.md](references/retrieval-ranking-and-filtering.md) | Formula scoring, MMR, ACORN, RRF, relevance feedback, named-vector filtering |
| [text-search.md](references/text-search.md) | Multilingual tokenization, stop words, stemming, phrase search, match-any, ASCII folding |
| [collections-writes-and-multitenancy.md](references/collections-writes-and-multitenancy.md) | Strict mode, tenant shards, conditional writes, upsert modes, metadata, vector schema changes |
| [observability-auditing-and-ui.md](references/observability-auditing-and-ui.md) | Cluster telemetry, optimization status, memory, metrics, audit logs, Web UI |

## Upgrade constraints and changed defaults

### Preserve adjacent-minor compatibility

Do not skip a minor-version migration step. Before moving to a target minor,
bring every node to the latest patch of the immediately preceding minor.
This rule also applies to a single-node deployment because skipped steps can
skip required data migrations.

For example, before moving from 1.16 to 1.17, first reach 1.16.3. A 1.17 node
can interoperate with 1.16 during the rollout, but not with 1.15.

### Upgrade clients before servers

Upgrade Qdrant client SDKs first. The SDK compatibility window covers the
latest three server minor versions, which keeps the client usable while the
cluster is rolled forward.

### Check replication before promising zero downtime

Zero-downtime rolling upgrades require every collection to have replication
factor 2 or greater. Restart one node at a time. A single-node cluster, or
even one collection with replication factor 1, requires a short outage.

For Kubernetes installations managed by the Qdrant chart, a Helm release
upgrade rolls the StatefulSet automatically:

```bash
helm upgrade qdrant qdrant/qdrant --version <target-version> -n <namespace>
```

### Account for the storage default

New deployments use Gridstore as the embedded storage backend. When upgrading
an older deployment, prefer adjacent-version upgrades even when APIs and
indexes do not otherwise require migration.

See
[deployment-upgrades-and-operations.md](references/deployment-upgrades-and-operations.md)
before planning a rollout or tuning ingestion back pressure.

## Strict-mode guardrails

Configure `strict_mode_config` at collection level to reject costly requests
instead of allowing them to consume unbounded resources. Useful limits cover:

- unindexed filtering during retrieval;
- oversized batches and payloads;
- too many filter conditions or payload indexes;
- excessive timeouts, `hnsw_ef`, or oversampling;
- high resident-memory pressure;
- too many queries in one batch-search request.

New collections enable strict mode by default. The `enabled` flag is dynamic,
and existing collections can be changed with `PATCH`. A rejected operation
returns a client error identifying the exceeded limit.

```http
PATCH /collections/{collection_name}
{
  "strict_mode_config": {
    "enabled": true,
    "max_resident_memory_percent": 90,
    "search_max_batchsize": 64
  }
}
```

Keep service limits and client batch sizes aligned. Read
[collections-writes-and-multitenancy.md](references/collections-writes-and-multitenancy.md)
for the complete set of collection and write controls.

## Retrieval and reranking choices

Choose the query mechanism from the retrieval goal:

| Goal | Mechanism |
| --- | --- |
| Mix vector relevance with business signals | Formula-based score boosting |
| Increase diversity among nearest neighbors | Maximal Marginal Relevance |
| Improve HNSW traversal under difficult filters | Per-query ACORN |
| Tune rank fusion decay | Configurable RRF `k` |
| Give contributing rankers different influence | Weighted RRF |
| Refine similarity from positive/negative context pairs | Relevance Feedback Query |
| Reduce replica tail latency | Delayed read fan-out |

Formula queries rescore a prefetched candidate set. They can combine `$score`,
payload conditions, numeric expressions, datetime decay, geographic distance,
and fallback `defaults`.

MMR uses `diversity` from `0.0` for relevance to `1.0` for diversity.
`candidates_limit` bounds the initial pool. ACORN is query-time only and needs
no index rebuild, but adds runtime work, so enable it only for filtered queries
that suffer from disconnected HNSW neighborhoods.

Open
[retrieval-ranking-and-filtering.md](references/retrieval-ranking-and-filtering.md)
for request shapes and the distinctions between these mechanisms.

## Full-text index choices

Set text behavior when creating the payload index:

- `tokenizer: "multilingual"` handles languages such as Chinese and Japanese
  that do not rely on whitespace word boundaries.
- `stopwords` removes configured common words during indexing and querying.
- a Snowball `stemmer` normalizes grammatical variants for a chosen language.
- `phrase_matching: true` builds the additional structure required by
  `match.phrase`; it cannot be added merely as a query option.
- `ascii_folding: true` makes unaccented query text match accented text.

Use `match.text_any` when any token in a multi-term string may match. It
replaces a client-generated set of `should` conditions.

Read [text-search.md](references/text-search.md) before defining or migrating a
full-text payload index.

## Indexing, storage, and quantization

Use the index and compression features according to workload:

- Vulkan GPU images can build HNSW indexes on supported GPUs, including
  concurrent indexing across multiple GPUs.
- Incremental HNSW extends the graph for upserts; deletes and updates can still
  require a full rebuild.
- 1.5-bit and 2-bit binary modes trade compression for accuracy, with 2-bit
  explicitly representing zero.
- Asymmetric quantization can combine binary stored vectors with
  scalar-quantized query vectors.
- TurboQuant rotates vectors before compression and supports cosine, dot
  product, and L2 distance.
- `inline_storage: true` places vector data in HNSW nodes to reduce random disk
  reads, but requires quantization and consumes more storage.

Do not assume every payload index should affect HNSW. Disable participation for
fields that are not used with dense-vector queries to avoid unnecessary graph
edges.

See
[indexing-storage-and-quantization.md](references/indexing-storage-and-quantization.md)
for the compression ratios and operating tradeoffs.

## Collection evolution and multitenancy

For tiered multitenancy, keep small tenants in a shared fallback shard and
create user-defined dedicated shards for large tenants. Continue sending a
shard key selector; Qdrant resolves it to the dedicated shard when present and
otherwise uses the fallback. Promote a tenant through shard transfer without
changing application routing.

Protect writes with the narrowest available semantic:

- attach an update filter to reject stale conditional updates;
- use insert-only mode when a create must not overwrite an existing point;
- use update-only mode when an update must not create a missing point.

Named vectors can be added to or removed from an existing schema. For embedding
migrations, add the new vector, backfill it, switch reads, and only then remove
the old vector. Collections can also carry custom metadata, and the shard-key
listing operation can discover the current custom sharding layout.

## Operations, telemetry, and auditing

Use `/cluster/telemetry` for a cluster-wide view rather than polling every
peer. Use `/collections/{collection_name}/optimizations` for active and past
optimization work. Inspect the collection Memory view or API for disk, RAM,
and page-cache use by vectors, payload, and indexes.

For per-collection request metrics:

```http
GET /metrics?per_collection=true
```

This adds a `collection` label to REST and gRPC response counters, failures,
and duration series. Consider the label cardinality before enabling it in a
large environment.

Audit protected operations when authentication or authorization is enabled.
Query audit entries across all nodes, filter them by time or field value, and
correlate requests by supplying `x-request-id`, `x-tracing-id`, or
`traceparent`.

Read
[observability-auditing-and-ui.md](references/observability-auditing-and-ui.md)
for endpoint scope, UI surfaces, and request-scoped inference credentials.

## Working method

1. Identify the deployed server version, topology, replication factors,
   collection schema, and client SDK version.
2. Open the reference for the task instead of inferring option names.
3. Distinguish collection-creation settings from dynamic settings and
   per-query controls.
4. For schema or index changes, state whether backfill, rebuilding, or extra
   storage is required.
5. For upgrades, validate every intermediate minor and every collection's
   replication factor before describing availability.
6. For performance features, describe both the intended workload and the
   additional CPU, memory, storage, or latency cost.
7. Prefer observed collection and cluster behavior over assumptions, then use
   these references to explain the relevant compatibility rule.
