# Upgrades and Operations

## Plan an Upgrade

### Follow adjacent minors

Before advancing to a new minor, bring every node to the latest patch release
of the immediately preceding minor. A 1.17 node can interoperate with 1.16, but
not directly with 1.15; for example, a 1.15 deployment must first reach 1.16.3
before moving to 1.17.

This constraint applies to a single-node deployment as well as a cluster.
Skipping the intervening minor can skip required data migrations.

### Upgrade clients first

Upgrade Qdrant client SDKs before the cluster. The SDKs are tested for backward
compatibility with the latest three Qdrant minor versions, so the newer client
can remain compatible while servers are rolled forward.

### Decide whether zero downtime is possible

Restart cluster nodes one at a time. Zero downtime requires every collection to
have a replication factor of at least 2. A single-node cluster, or any
collection with replication factor 1, requires a short restart outage.

### Upgrade a Helm deployment

Upgrading the Helm release automatically rolls the Qdrant StatefulSet:

```bash
helm upgrade qdrant qdrant/qdrant --version <target-version> -n <namespace>
```

Confirm replica factors and the adjacent-minor path before invoking the
rollout.

## Choose Storage and Indexing Infrastructure

### Account for the Gridstore default

New deployments use Gridstore instead of RocksDB as the default embedded
storage backend (since 1.15.0). That release has no major API- or
index-breaking upgrade change, but upgrades from older releases should still
proceed one version at a time.

### Build HNSW indexes on GPUs

Qdrant can build HNSW indexes on modern Vulkan-capable GPUs (since 1.13.0).
The on-premises capability is distributed through preconfigured GPU container
images and supports:

- major GPU hardware families;
- concurrent segment indexing across multiple GPUs;
- every Qdrant quantization option and data type.

Check startup and indexing logs to confirm GPU detection and actual use.

### Understand incremental HNSW work

Existing HNSW graphs can be extended when new points are added (since 1.14.0).
The incremental path applies only to upserts of new points in its initial
implementation. Deletes and updates still cause a full rebuild.

### Reduce random reads with inline storage

Set `hnsw_config.inline_storage` to `true` to store vector data in HNSW nodes
(since 1.16.0):

```json
{
  "hnsw_config": {
    "inline_storage": true
  }
}
```

Quantization must also be enabled. Inline storage consumes additional storage,
but reduces random disk reads and enables implicit rescoring from the original
vector stored with the node.

## Manage Write and Read Pressure

### Let shard queues apply back pressure

Shards queue up to one million pending changes and apply back pressure when the
queue fills (since 1.17.0). This bounds load during heavy ingestion and replica
recovery.

For latency-sensitive indexed-only search, enable the `prevent_unoptimized`
optimizer setting. It throttles writes toward the indexing rate so large
unoptimized segments do not accumulate.

### Hedge only slow replica reads

Delayed read fan-out sends a second request to another replica after the first
replica exceeds a configured latency threshold (since 1.17.0). Qdrant returns
whichever response arrives first. This targets tail latency without querying
multiple replicas immediately for every read.

## Observe the Cluster

### Aggregate telemetry

Use `GET /cluster/telemetry` for information from all peers (since 1.17.0).
Unlike polling each peer's `/telemetry` endpoint, it exposes cluster-wide
activity in one response, including:

- leader elections;
- resharding;
- shard transfers.

### Monitor segment optimization

Use `/collections/{collection_name}/optimizations` to inspect cluster-wide
optimization status and the details of current and past operations (since
1.17.0). The Web UI **Optimizations** tab shows the same area as status,
timelines, and per-cycle task durations.

### Search points in the Web UI

The point-search interface can find points similar to a selected point, apply
payload filters, or locate a point directly by ID (since 1.17.0).

### Inspect collection memory

Collection memory reporting breaks down disk, RAM, and OS page-cache use by
component, including vectors, payload, and indexes (since 1.18.0). Values are
aggregated across the cluster and are available through an API and the
collection detail page's **Memory** tab.

### Add per-collection response metrics

Pass `per_collection=true` to the metrics endpoint (since 1.18.0):

```http
GET /metrics?per_collection=true
```

This adds a `collection` label to `rest_responses_*` and `grpc_responses_*`,
covering per-collection request counts, failures, and response durations.
