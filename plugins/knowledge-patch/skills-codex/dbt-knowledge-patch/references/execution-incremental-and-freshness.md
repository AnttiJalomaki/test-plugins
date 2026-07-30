# Execution, Incremental Models, and Freshness

Use this reference for time-windowed execution, retries, reduced-data runs, and
freshness-driven orchestration. Relevant extraction sections: 1.9-guides,
1.9.0, 1.10.0, 1.11.0, and 1.12.0.

## Microbatch incremental strategy

The `microbatch` incremental strategy splits large time-series models into
independently replaceable time batches. dbt auto-filters direct `ref()` and
`source()` inputs that declare `event_time`; write model SQL for one batch and
do not add an `is_incremental()` filter merely to bound time.

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

`begin`, `event_time`, and `batch_size` are required. Valid batch sizes are
`hour`, `day`, `month`, and `year`. `lookback` defaults to one batch.
`concurrent_batches: true` or `false` overrides automatic parallelism
detection.

Configure each direct parent separately:

```yaml
models:
  - name: stg_events
    config:
      event_time: my_time_field
```

An unconfigured parent is fully scanned for every batch. To opt a configured
parent out of automatic filtering, deliberately render it with
`ref('stg_events').render()`.

Adapter requirements differ: PostgreSQL additionally requires `unique_key`,
while Spark and BigQuery require `partition_by`.

## Backfills, hooks, and retry

A backfill needs both event-time bounds:

```bash
dbt run --event-time-start "2024-09-01" --event-time-end "2024-09-04"
```

`event_time`, `begin`, and both CLI bounds are interpreted as UTC. `dbt retry`
reruns only failed batches. It honors `--threads`; from 1.10.20, it recomputes
batches using the original invocation time instead of the retry time.

Microbatch model Jinja has a `batch` context object. Pre-hooks run only for the
first batch and post-hooks only for the last batch.

A custom microbatch strategy macro must be paired with this project flag:

```yaml
flags:
  require_batched_execution_for_custom_microbatch_strategy: true
```

## Sample mode

Core 1.10 introduces sample mode and enables it for `dbt build`. The finalized
CLI folds the separate sample-window parameter into `--sample`. Sampling
extends to referenced seeds and through snapshot dependency graphs, so a
sampled build can reduce upstream input consistently rather than sampling only
the selected model.

## Empty and schema-only execution

Snapshots accept `--empty`. Jinja can detect an empty run:

```jinja
{% if flags.EMPTY %}
  -- schema-only execution
{% endif %}
```

Core 1.12 adds empty seed relations:

```bash
dbt seed --empty --select customers
```

This creates seed tables without inserting rows, which is useful for preparing
schemas or ancestors for unit tests. Managed-function unit tests commonly use:

```bash
dbt build --select "+my_model_to_test" --empty
```

## Source freshness

Source freshness can live under `config`; explicitly configured null freshness
values are preserved (since 1.9.0).

```yaml
sources:
  - name: raw
    config:
      freshness:
        warn_after: {count: 12, period: hour}
```

Source and table configs accept either `loaded_at_field` or
`loaded_at_query`. A query can calculate freshness when there is no suitable
column expression:

```yaml
sources:
  - name: raw
    tables:
      - name: events
        config:
          loaded_at_query: "select max(_loaded_at) from raw.events"
```

From Core 1.10, `dbt source freshness` runs project hooks by default through
the matured `source_freshness_run_project_hooks` behavior. Setting the flag to
`false` temporarily retains legacy behavior but warns; Core 2.0 removes the
flag.

## Model freshness

Model freshness for adaptive jobs is config-only. In 1.10 it is skipped
without `build_after`, and `build_after` requires both `count` and `period`:

```yaml
models:
  - name: orders
    config:
      freshness:
        build_after:
          count: 12
          period: hour
```

Core 1.11 adds update-driven freshness. An `updates_on` trigger can stand alone
without `count` or `period`:

```yaml
models:
  - name: orders
    config:
      freshness:
        build_after:
          updates_on: any
```

The later form is additive behavior, not a reason to remove time-based
freshness from existing models.

## Configurable input limits

Core 1.12 makes the `MAXIMUM_SEED_SIZE_MIB` seed-size limit configurable. The
new `--sqlparse` option configures SQL-parser limits, replacing the need to pin
a particular `sqlparse` release solely to control parser limits.
