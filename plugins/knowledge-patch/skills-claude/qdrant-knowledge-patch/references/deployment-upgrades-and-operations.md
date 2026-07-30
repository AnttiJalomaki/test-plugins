# Deployment, Upgrades, and Operations

## Safe upgrade sequence

### Do not skip adjacent minors

Before advancing to a new minor, every node must run the latest patch of the
immediately preceding minor (`upgrade-guide`). Compatibility is adjacent: a
1.17 node can coexist with 1.16 during a rollout, but not with 1.15. For
example, upgrade 1.15 to 1.16.3 before introducing 1.17.

This constraint applies to single-node deployments too. Skipping the
intermediate release can skip required data migrations.

### Upgrade the client SDK first

Upgrade Qdrant client SDKs before upgrading the cluster (`upgrade-guide`).
Client SDKs are tested for backward compatibility with the latest three Qdrant
minor versions, so the newer client remains compatible while servers are
rolled forward.

### Verify replication before a rolling restart

A rollout is zero-downtime only when every collection has replication factor
2 or greater and nodes restart one at a time (`upgrade-guide`). A single-node
cluster or any collection with replication factor 1 requires a short restart
outage.

### Helm upgrades

Upgrading the Helm release automatically rolls the Qdrant StatefulSet
(`upgrade-guide`):

```bash
helm upgrade qdrant qdrant/qdrant --version <target-version> -n <namespace>
```

Apply the same adjacent-minor and replication checks before invoking Helm.

## Storage default

New deployments use Gridstore instead of RocksDB as the default embedded
storage backend (since 1.15.0). That release has no major API- or
index-breaking upgrade changes, but upgrades from older releases should still
proceed one version at a time.

## Write back pressure

Each shard can queue up to one million pending changes and applies back
pressure when the queue fills (since 1.17.0). This prevents heavy ingestion or
replica recovery from creating runaway load.

For latency-sensitive searches that request indexed data only, use the
`prevent_unoptimized` optimizer setting to throttle writes toward the indexing
rate. The goal is to prevent large unoptimized segments from accumulating;
account for the resulting ingestion slowdown when setting client retry and
timeout behavior.
