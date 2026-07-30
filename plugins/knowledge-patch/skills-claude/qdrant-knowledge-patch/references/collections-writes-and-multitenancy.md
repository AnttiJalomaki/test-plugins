# Collections, Writes, and Multitenancy

## Strict mode

Collection-level `strict_mode_config` rejects expensive operations that cross
configured limits (since 1.13.0). Guardrails include unindexed filtering,
batch and payload sizes, filter-condition counts, timeouts, `hnsw_ef`, and
oversampling. The client error identifies the limit that was exceeded.

New collections enable strict mode by default. `enabled` is a dynamic toggle,
and an existing collection can be updated with `PATCH`.

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
  resident memory exceeds the configured percentage of total system memory.
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

## Tiered multitenancy

A multitenant collection can combine a shared fallback shard for small tenants
with user-defined dedicated shards for large tenants (since 1.16.0). Requests
continue to supply a shard key selector. Qdrant routes them to a matching
dedicated shard when one exists and to the fallback shard otherwise.

Promote a growing tenant from the fallback to a dedicated shard using the
shard-transfer mechanism. Reads and writes continue during promotion, so the
application does not need separate routing for shared and dedicated tenants.

An API operation lists all user-defined shard keys (since 1.17.0). Use it to
discover a collection's current custom sharding layout rather than relying on
an application-side inventory.

## Concurrency-safe and intention-revealing writes

### Conditional updates

A point update can carry a filter that must match before Qdrant applies the
write (since 1.16.0). The update is rejected when the condition fails.

Use an expected version, synchronized timestamp, or other monotonically
increasing payload value in the filter to prevent stale writers and background
re-embedding jobs from overwriting newer data.

### Insert-only and update-only upserts

Upserts accept an update mode that restricts an operation to inserting a new
point or updating an existing point (since 1.17.0). Insert-only prevents an
intended create from overwriting a point. Update-only prevents an intended
update from silently creating one.

## Collection metadata

Collections can carry custom metadata (since 1.16.0). Keep operational or
application descriptors there when they belong to the collection rather than
to individual point payloads.

## Named-vector schema evolution

Named vectors can be added to or removed from an existing collection schema
without recreating and re-ingesting the collection (since 1.18.0).

For an embedding migration:

1. Add the new named-vector field.
2. Populate it in the background.
3. Move queries to the new vector.
4. Remove the old vector only after migration is complete.
