---
name: qdrant-knowledge-patch
description: Qdrant
version: 1.18.0
license: MIT
metadata:
  author: Nevaberry
---


# Qdrant Knowledge Patch

Use this skill when designing, upgrading, operating, or developing against
Qdrant. Start with the compatibility checks, then open the reference that
matches the collection, query, indexing, observability, or security task.

## When to Load This Skill

Load this skill for work involving:

- rolling or single-node Qdrant upgrades;
- collection schemas, named vectors, strict mode, or custom sharding;
- dense, hybrid, filtered, full-text, or diversified retrieval;
- HNSW indexing, quantization, storage, or write optimization;
- cluster telemetry, memory, metrics, or optimization diagnosis;
- audit trails or request-scoped inference credentials.

## Breaking Changes and Upgrade Checks

### Preserve the adjacent-minor path

Do not skip a minor release. Every node must first run the latest patch of the
immediately preceding minor, including in single-node deployments. Skipping
that step can bypass required data migrations.

Upgrade client SDKs before servers. During a clustered rollout, restart one
node at a time. Zero downtime requires every collection to have at least two
replicas; otherwise plan a restart outage.

For Kubernetes installations, a Helm release upgrade rolls the StatefulSet
automatically. Verify the replica requirement before relying on that rollout
for uninterrupted service.

### Account for the storage default

New deployments use Gridstore as the embedded storage backend. The change does
not itself introduce a major API or index break, but it does not relax the
adjacent-minor upgrade requirement.

### Expect strict-mode rejections

New collections enable strict mode by default. Requests can fail with a client
error when they exceed collection guardrails for filtering, request size,
query complexity, memory, or batch search. Treat these failures as policy
signals and inspect `strict_mode_config` before retrying with more work.

### Know which index changes rebuild

Incremental HNSW extension applies to new-point upserts. Deletes and updates
still trigger a full graph rebuild. Plan mutation-heavy workloads and
optimization monitoring around that distinction.

### Enable phrase support before querying it

Exact phrase search depends on an extra full-text index structure. Set
`phrase_matching: true` when creating the payload index; adding
`match.phrase` only at query time is insufficient.

### Satisfy inline-storage prerequisites

`hnsw_config.inline_storage: true` requires quantization. It trades additional
storage for fewer random disk reads and can rescore from the original vector
kept with each HNSW node.

## Reference Index

| Reference | Topics |
| --- | --- |
| [`references/upgrades-and-operations.md`](references/upgrades-and-operations.md) | Upgrade sequencing, GPU indexing, storage, write pressure, replicas, telemetry, optimizations, memory, metrics, and Web UI |
| [`references/vector-search-and-ranking.md`](references/vector-search-and-ranking.md) | Vector-presence filtering, formulas, HNSW behavior, quantization, ACORN, MMR, feedback, RRF, and TurboQuant |
| [`references/full-text-search.md`](references/full-text-search.md) | Multilingual tokenization, stop words, stemming, phrase search, match-any queries, and ASCII folding |
| [`references/collections-and-writes.md`](references/collections-and-writes.md) | Strict mode, tiered tenants, conditional writes, metadata, shard keys, upsert modes, and named-vector schema changes |
| [`references/auditing-and-inference.md`](references/auditing-and-inference.md) | Protected-operation auditing, cluster-wide audit queries, tracing correlation, and request-scoped inference credentials |

## Quick Reference: Collection Safety

### Strict mode

Use collection strict mode to bound expensive requests. Relevant controls
include:

- unindexed filtering for retrieval;
- oversized batches and payloads;
- excessive filter conditions;
- timeout, `hnsw_ef`, and oversampling limits;
- the maximum number of payload indexes;
- resident-memory and batch-search limits.

`enabled` is a dynamic toggle, and existing collections can be changed with
`PATCH`. A rejected request identifies the exceeded limit.

### Conditional writes

Attach an update filter when a write must only apply to the state the caller
observed. An expected version, synchronized timestamp, or monotonic payload
value can protect newer content from stale writers and background
re-embedding.

Use insert-only or update-only upsert modes when the invariant is point
existence rather than payload state. Insert-only prevents overwrite; update-only
prevents accidental creation.

### Named-vector migrations

Add a new named vector to the existing schema, populate it in the background,
move reads to it, and then remove the old field. Collection recreation and
full re-ingestion are no longer required for this migration.

### Tenant placement

Use a shared fallback shard for small tenants and dedicated user-defined shards
for larger tenants. Continue sending the shard key selector; Qdrant chooses the
dedicated shard when present and the fallback otherwise.

Promotion uses shard transfer while reads and writes continue. Application
routing does not need separate paths for shared and dedicated placement.

## Quick Reference: Retrieval and Ranking

### Filter and index deliberately

- Use `has_vector` to select points that contain a particular named vector.
- Turn on ACORN only for filtered HNSW queries affected by low-selectivity
  filters; it adds query-time graph exploration and runtime cost.
- Exclude payload field indexes from HNSW participation when they are not used
  with dense-vector queries, avoiding unnecessary graph edges.
- Use `prevent_unoptimized` when indexed-only query latency matters more than
  unrestricted ingestion throughput.

### Choose a reranking mechanism

- Formula queries combine `$score`, payload conditions, numeric expressions,
  decay functions, and fallback values after prefetch.
- MMR balances relevance and diversity through `diversity` and
  `candidates_limit`.
- Relevance feedback adjusts retrieval from more-relevant and less-relevant
  context pairs.
- RRF supports a configurable `k`; weighted RRF also controls each contributing
  query's influence.

### Choose quantization by workload

- Multi-bit binary modes provide intermediate compression and accuracy options.
- Asymmetric quantization can pair binary stored vectors with scalar-quantized
  queries to improve precision while retaining near-binary storage.
- TurboQuant rotates vectors before compression, avoiding a centered-vector
  assumption and supporting cosine, dot-product, and L2 distance.

See the vector-search reference for the exact tradeoffs and constraints.

## Quick Reference: Full-Text Search

Configure text behavior when creating the payload index:

- `tokenizer: "multilingual"` handles languages without whitespace word
  boundaries;
- `stopwords` removes configured common words;
- a Snowball `stemmer` normalizes related grammatical forms;
- `phrase_matching: true` builds support for ordered phrase queries;
- `ascii_folding: true` normalizes diacritics in indexed text and search terms.

At query time, use `match.text_any` to match at least one token from a
multi-term string without building a client-side group of `should` conditions.

## Quick Reference: Operations and Observability

### Control ingestion pressure

Each shard queues pending changes and applies back pressure when its queue
fills. `prevent_unoptimized` further throttles writes toward the indexing rate
so large unoptimized segments do not accumulate.

### Reduce tail latency

Configure delayed read fan-out to send a second request to another replica only
after the first crosses a latency threshold. Qdrant uses whichever response
arrives first, avoiding eager fan-out on normal reads.

### Inspect cluster state

- `/cluster/telemetry` aggregates peer activity such as leader elections,
  resharding, and shard transfers.
- `/collections/{collection_name}/optimizations` shows cluster-wide status and
  current and past optimization work.
- Collection memory reporting breaks disk, RAM, and OS page-cache usage down
  by vectors, payload, and indexes across the cluster.
- `/metrics?per_collection=true` adds collection labels to REST and gRPC
  response metrics.

The Web UI exposes optimization timelines, component memory, and point lookup
by similarity, payload filter, or ID.

## Quick Reference: Auditing and Credentials

Enable audit logging when protected API activity needs an operational record.
Use the query endpoint to aggregate entries across cluster nodes and filter by
time or recorded field values.

Propagate `x-request-id`, `x-tracing-id`, or `traceparent` to correlate audit
entries with application and distributed traces. Supply external inference
credentials in request headers when keys must vary per inference request.

## Task Checklist

Before changing a Qdrant deployment or application:

1. Identify whether the task changes server versions, collection state, or only
   query behavior.
2. For an upgrade, confirm the adjacent-minor path, SDK-first order, replica
   factors, and one-node-at-a-time rollout.
3. For a collection change, inspect strict mode, quantization, HNSW settings,
   shard keys, and named-vector schema.
4. For retrieval changes, distinguish index-time prerequisites from query-time
   options.
5. For write changes, choose conditional, insert-only, or update-only semantics
   explicitly.
6. For performance work, inspect optimization, memory, and telemetry data
   before changing limits.
7. For protected operations, preserve authentication context and tracing IDs
   in the audit path.
8. Open the relevant reference and apply its exact constraints and examples.
