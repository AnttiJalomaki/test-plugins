# Assets, partitions, and state

## Typed asset-event access

String keys no longer work with `inlet_events`, `outlet_events`, or
`triggering_asset_events` in 3.0.0. Address an asset or alias with `Asset`,
`AssetAlias`, or `Asset.ref`, or use the lookup helpers:

```python
outlet_events[Asset.ref(name="myasset")]
outlet_events[AssetAlias(name="myalias")]
outlet_events.for_asset(name="myasset")
outlet_events.for_asset_alias(name="myalias")
```

Use `create_asset_aliases()` to define aliases that are shared across Dag
files.

Asset API responses use `scheduled_dags` instead of `consuming_dags` as of
3.1.0. This field means Dags that put the asset in their `schedule`, not every
Dag that otherwise consumes or references it.

## Partition-aware asset scheduling

Since 3.2.0, asset-aware scheduling can trigger downstream Dags only for the
updated partition rather than for the entire asset.

- `AllowedKeyMapper` validates partition keys.
- `ChainMapper` composes multiple mappings.
- Temporal mapper names use `StartOfXXXMapper`, not `ToXXXMapper`.
- Inlet events can be lazily filtered by time, ordering, and limit.
- Listeners can receive asset-emission events.

## Fan-out, rollup, and runtime keys

Airflow 3.3.0 expands partition mapping with:

- `RollupMapper` for aggregating upstream partitions.
- `FanOutMapper` for forward or backward expansion.
- Categorical `FixedKeyMapper`.
- `SegmentWindow` mappings over temporal windows.
- `WaitForAll` and `MinimumCount(n)` readiness behavior.
- `PartitionedAtRuntime` for assigning partition keys when a Dag run begins.

Use `[scheduler] partition_mapper_max_downstream_keys` as the global cap on
expanded downstream keys. A mapper can override the cap where the fan-out is
intentionally larger.

## Clearing and backfilling partitions

In 3.3.0, the REST API adds `clearPartitions` and bulk
`/dags/{dag_id}/clearDagRuns`. Select targets with `partition_key` and
`partition_date` windows. CLI clear and backfill operations also accept
partition ranges.

Producer partition dates and task-emitted partition keys propagate through
asset events to partitioned consumers, so preserve both when building custom
event integrations.

## Task and asset state stores

Airflow 3.3.0 adds Task SDK accessors `task_state_store` and
`asset_state_store`. They persist JSON state and support:

- `get`, `set`, `delete`, and `clear`;
- expiration and retention;
- optional `clear_on_success`;
- Core API and Execution API access;
- asset-state access from triggers.

Task state can survive both retries and runs. Storage defaults to the metadata
database; `[workers] state_store_backend` selects a worker-side backend.
Retention garbage collection and row-size limits are configurable.

`task_state_store.clear()` no longer accepts `all_map_indices`; select the
supported scope explicitly.

## Team-scoped asset access

The multi-team Asset SDK changes again in 3.3.0: replace
`allow_producer_teams` with `access_control`. `AssetAccessControl` adds
`consumer_teams` and `allow_global`.

Asset queries are team-scoped, as is the XCom Execution API. Pool scheduling
also enforces team ownership, and triggerers can be assigned and filtered by
team. Design partition and state-store access around those boundaries rather
than relying only on UI permissions.
