# Collections and Writes

## Bound Collection Work with Strict Mode

Collection-level `strict_mode_config` rejects expensive operations that exceed
configured limits (since 1.13.0). Guardrails include:

- unindexed filtering during retrieval;
- oversized batches or payloads;
- excessive filter conditions;
- timeouts;
- `hnsw_ef`;
- oversampling.

Rejected requests return a client error identifying the exceeded limit. New
collections enable strict mode by default. `enabled` is a dynamic toggle, and
existing collections can be updated with `PATCH`.

```http
PUT /collections/{collection_name}
{
  "strict_mode_config": {
    "enabled": true,
    "unindexed_filtering_retrieve": true
  }
}
```

Strict mode can also cap the number of payload indexes (since 1.16.0).

Two additional guardrails are available (since 1.18.0):

- `max_resident_memory_percent` rejects memory-consuming writes after process
  resident memory exceeds the configured percentage of total system memory;
- `search_max_batchsize` limits the number of queries in one batch-search
  request.

```http
PATCH /collections/{collection_name}
{
  "strict_mode_config": {
    "max_resident_memory_percent": 90,
    "search_max_batchsize": 64
  }
}
```

## Place and Discover Tenants

### Combine fallback and dedicated shards

A multitenant collection can keep small tenants on a shared fallback shard and
large tenants on user-defined dedicated shards (since 1.16.0). Requests provide
a shard key selector. Qdrant routes to the tenant's dedicated shard when it
exists and otherwise uses the fallback.

Promote a growing tenant with Qdrant's shard-transfer mechanism. Reads and
writes continue during promotion, so application code does not need separate
routing logic for shared and dedicated tenants.

### List the current shard keys

Use the shard-key listing API to retrieve all user-defined shard keys (since
1.17.0). This lets applications and operators discover the collection's
current custom sharding layout.

## Protect Point Writes

### Make an update conditional

Attach an update filter that must match before Qdrant applies a point update
(since 1.16.0). Qdrant rejects the update when the condition fails.

Use an expected version field, synchronized timestamp, or other monotonically
increasing payload value to prevent stale writers or background re-embedding
jobs from overwriting newer changes.

### Separate insert and update intent

Choose an insert-only or update-only upsert mode (since 1.17.0). Insert-only
prevents an intended create from overwriting an existing point. Update-only
prevents an intended update from silently creating a missing point.

## Evolve Collection Metadata and Vectors

### Store collection metadata

Collections can carry custom metadata (since 1.16.0). Keep collection-wide
application context there rather than duplicating it on every point.

### Change named-vector schemas in place

Named vectors can be added to or removed from an existing collection schema
(since 1.18.0) without recreating and re-ingesting the collection.

For an embedding migration:

1. Add the new vector field.
2. Populate it in the background.
3. Move reads to the new field.
4. Remove the old vector field.
