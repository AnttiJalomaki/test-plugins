# Incremental Models, Snapshots, and Freshness

## Microbatch incremental models

The `1.9-guides` batch introduces the `microbatch` incremental strategy for
large time-series data. dbt divides execution into independently replaceable
time batches and automatically filters direct `ref()` and `source()` inputs
that have `event_time`. Model SQL describes one batch and does not need an
`is_incremental()` filter.

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='microbatch',
    event_time='event_occurred_at',
    begin='2020-01-01',
    batch_size='day',
    lookback=3,
    full_refresh=false
) }}

select * from {{ ref('stg_events') }}
```

Configure `event_time` separately on each direct parent that should be
filtered. An unconfigured parent is fully scanned for every batch. Calling
`.render()` deliberately opts a configured parent out of automatic filtering:

```yaml
models:
  - name: stg_events
    config:
      event_time: my_time_field
```

Required settings are `begin`, `event_time`, and `batch_size`. Valid batch
sizes are `hour`, `day`, `month`, and `year`. `lookback` defaults to one batch;
`concurrent_batches: true` or `false` overrides automatic parallelism
detection. PostgreSQL also requires `unique_key`, while Spark and BigQuery
require `partition_by`.

Backfills require both bounds:

```bash
dbt run --event-time-start "2024-09-01" --event-time-end "2024-09-04"
```

`event_time`, `begin`, and both command bounds are interpreted as UTC. A custom
microbatch strategy macro additionally requires this project flag:

```yaml
flags:
  require_batched_execution_for_custom_microbatch_strategy: true
```

Core 1.10.0 adds the `batch` Jinja context. For microbatch models, pre-hooks run
only on the first batch and post-hooks only on the last. `dbt retry` reruns only
failed batches and honors `--threads`. Starting in 1.10.20, retry recomputes
batches from the original invocation time rather than the retry time.

## Sample and empty execution

Core 1.10 introduces sample mode and enables it for `dbt build`. The final CLI
folds the earlier separate `--sample-window` input into `--sample`. Sampling
extends to referenced seeds and through snapshot dependency graphs.

Snapshots accept `--empty`, and Jinja can inspect the mode through `flags.EMPTY`
(from `1.9.0`):

```jinja
{% if flags.EMPTY %}
  -- schema-only execution
{% endif %}
```

Core 1.12.0 adds `dbt seed --empty`, which creates seed tables without loading
rows:

```bash
dbt seed --empty --select customers
```

## Snapshot hard deletes

Snapshots support `hard_deletes` modes `ignore` (default), `invalidate`, and
`new_record` (from `1.9-guides`). `invalidate` closes a deleted row by setting
`dbt_valid_to`; `new_record` adds a deletion row and the `dbt_is_deleted`
metadata column.

```yaml
snapshots:
  - name: my_snapshot
    config:
      unique_key: id
      strategy: timestamp
      updated_at: updated_at
      hard_deletes: new_record
```

The legacy `invalidate_hard_deletes` remains accepted but cannot be combined
with `hard_deletes`. Existing tables are not migrated automatically. Migrate
their schema and data before switching modes, or apply the new setting only to
new snapshots. PostgreSQL, BigQuery, Snowflake, and Redshift adapters support
this configuration.

## Query-driven source freshness

Core 1.10.0 allows source and table configs to use `loaded_at_field` or
`loaded_at_query`. A query can calculate the latest load time directly:

```yaml
sources:
  - name: raw
    tables:
      - name: events
        config:
          loaded_at_query: "select max(_loaded_at) from raw.events"
```

Source freshness can also live under `config`, and explicit null freshness
values are preserved (from `1.9.0`):

```yaml
sources:
  - name: raw
    config:
      freshness:
        warn_after: {count: 12, period: hour}
```

## Model freshness

Model freshness for adaptive jobs is config-only. In Core 1.10.0, it is skipped
without `build_after`; a time-based trigger requires both `count` and `period`:

```yaml
models:
  - name: orders
    config:
      freshness:
        build_after:
          count: 12
          period: hour
```

Core 1.11.0 also permits update-driven freshness without `count` or `period`:

```yaml
models:
  - name: orders
    config:
      freshness:
        build_after:
          updates_on: any
```

## Latest-version relation pointers

Core 1.12.0 lets versioned models create an unversioned relation pointer such
as `dim_customers` for the latest version. Enable pointers project-wide with
`latest_version_pointer_enabled_by_default` or per model with
`latest_version_pointer`.

```yaml
flags:
  latest_version_pointer_enabled_by_default: true
```

Collision checks respect identifier quoting and case. Unquoted floating model
versions such as `v: 4.5` are no longer silently dropped.
