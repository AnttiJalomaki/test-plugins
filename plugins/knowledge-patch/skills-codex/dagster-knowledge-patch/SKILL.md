---
name: dagster-knowledge-patch
description: Dagster
version: 1.13.0
license: MIT
metadata:
  author: Nevaberry
---


# Dagster Knowledge Patch

Use this skill when upgrading or writing Dagster projects, Components, asset
automation, deployment configuration, or integration code. Start with the
breaking-change checklist, then open the topic reference that matches the work.

## Reference index

| Reference | Topics |
|---|---|
| [upgrades-and-api.md](references/upgrades-and-api.md) | Breaking changes, deprecations, API lifecycle, freshness migration |
| [assets-and-automation.md](references/assets-and-automation.md) | Assets, checks, partitions, selections, backfills, automation |
| [components-and-cli.md](references/components-and-cli.md) | Components, templates, state, project scaffolding, `dg`, Dagster+ CLI |
| [operations-and-deployment.md](references/operations-and-deployment.md) | Coordination, executors, GraphQL, databases, Kubernetes, ECS, daemons |
| [data-integrations.md](references/data-integrations.md) | dbt, Airbyte, Fivetran, Sling, dlt, BI tools, warehouse IO managers |
| [platform-integrations.md](references/platform-integrations.md) | Databricks, cloud resources, Pipes, authentication, newer packages |

## Upgrade first

### Keep runs launching

The queued run coordinator is the default and requires a running Dagster
daemon. For deliberately immediate in-process launches, opt back into the
synchronous coordinator:

```yaml
run_coordinator:
  module: dagster.core.run_coordinator.sync_in_memory_run_coordinator
  class: SyncInMemoryRunCoordinator
```

Concurrency-key and pool blocking is also enabled by default. At op
granularity, a run can leave the queue once one op is runnable. At run
granularity, every pool used by the run needs a free slot.

### Replace removed asset APIs

- Rename `include_sources` to `include_external_assets` on
  `AssetSelection` APIs.
- Pass `AssetSpec` objects directly to `Definitions`; do not call removed
  `external_asset_from_spec` or `external_assets_from_specs`.
- Pass `deps` as a sequence, even for one dependency. Use `AssetDep` when a
  dependency carries configuration.
- Replace `Definitions.get_all_asset_specs()` with
  `Definitions.resolve_all_asset_specs()`.

```python
from dagster import AssetDep, AssetSpec, Definitions, asset

defs = Definitions(assets=[AssetSpec("external_table")])

@asset(deps=[AssetDep("upstream")])
def downstream(): ...
```

### Complete the freshness migration

Do not confuse the old and new policy types:

- The former policy became `LegacyFreshnessPolicy`; its deprecated alias is
  available from `dagster.deprecated`, not top-level `dagster`.
- Import the current `FreshnessPolicy` and `apply_freshness_policy` from
  top-level `dagster`.
- The freshness daemon evaluates policies by default. Set
  `freshness.enabled: false` only when automatic evaluation is unwanted.
- Replace `build_.*_freshness_checks` helpers and integration translator
  `get_freshness_policy` hooks with freshness policies.
- Remove legacy freshness and auto-observation arguments. Use
  `automation_condition` plus schedule- or sensor-based automation.

```yaml
freshness:
  enabled: false
```

### Update Component loading

- Replace deprecated, non-public `load_defs` with
  `load_from_defs_folder(path)`.
- Use `context.load_component(...)` and `context.build_defs(...)`. The
  compatibility names `load_component_at_path` and `build_defs_at_path` are
  removed.
- Put Sling and Airflow `post_processors` at the top level; the old
  `asset_post_processors` field is gone.
- Give `SlingReplicationCollectionComponent` its `connections` directly,
  rather than the old `sling` YAML field or Python `resource` argument.

### Update commands and runtime declarations

- Use `create-dagster project`; all `dagster project` commands are removed.
- Use `dg plus deploy start` instead of deprecated
  `dagster-cloud ci check`.
- Do not call removed `dg docs integrations` or `dg utils integrations`.
- Require Python 3.10 or newer. Declare `psycopg2-binary` directly when a
  PostgreSQL deployment depends on it.
- On MySQL, run `dagster instance migrate` for the `LongText` migrations.

## High-value asset patterns

### Return and load asset values

`MaterializeResult(value=...)` sends the value through the asset IO manager
and supports `MaterializeResult[T]`. A downstream asset can load a dependency
dynamically through `AssetExecutionContext.load_asset_value`.

```python
import dagster as dg

@dg.asset
def upstream() -> dg.MaterializeResult[int]:
    return dg.MaterializeResult(value=42)

@dg.asset(deps=[upstream])
def downstream(context: dg.AssetExecutionContext):
    return context.load_asset_value(dg.AssetKey("upstream"))
```

### Model virtual assets

Use preview `is_virtual=True` on `@asset` or `AssetSpec` for a view or derived
table that reflects upstream changes without explicit materialization.
Virtual assets participate in staleness, planning, and declarative automation.

```python
import dagster as dg

view = dg.AssetSpec("reporting_view", is_virtual=True)
```

### Use partition-aware checks

`@asset_check` and `AssetCheckSpec` accept `partitions_def`; it must match the
target asset's partition definition. Execution contexts expose
`multi_partition_key` in multi-partition runs. Ops can also yield
`AssetCheckEvaluation`, and `@asset` accepts success and failure `hooks`.

### Build richer selections

Selection expressions combine lineage traversal, attribute filters, and
Boolean logic. They work across Components YAML, the Asset Catalog, saved
selections, alerts, and insights. Useful attributes include `sensor:`,
`schedule:`, `job:`, `automation_type:`, and the `is:` type filter.

```text
sensor:daily_refresh
automation_type:schedule
is:materializable
group:"marketing/*"
partitions:"static"
```

Schedule and sensor selectors include assets in targeted jobs as well as
direct selections. Group names may contain `/` and support hierarchical
wildcards.

## Component starting point

Components are the default starting point for new projects. Declare them in
`defs.yaml` or implement typed Python `Component` subclasses. Use
`@template_var` to expose Python helpers to YAML, and use
`build_defs_for_component` outside a `defs` folder.

```yaml
deps:
  - "{{ load_component_at_path('dbt_ingest').asset_key_for_model('customers') }}"
```

For current template code, helpers live in explicit namespaces:

```jinja
{{ dg.AutomationCondition.on_missing() & dg.AutomationCondition.in_latest_time_window() }}
{{ dg.DailyPartitionsDefinition("2025-01-01") }}
{{ context.load_component("warehouse") }}
```

Use `dg scaffold`, `dg dev`, `dg launch`, `dg list`, `dg check`, and
`dg utils` for normal project work. Generated projects use a `src/` plus
`defs/` layout and local `dg` configuration.

## Operational checks

Before deploying:

1. Ensure the daemon is running when queued coordination, schedules, sensors,
   freshness, or asset automation is expected.
2. Check pool names and granularity. Pool names can contain any
   non-whitespace character.
3. Paginate `logsForRun` and `eventConnection` after 1,000 events by following
   the returned cursor.
4. Confirm database migrations and explicitly declared database drivers.
5. Check state-backed integration storage; local filesystem state is now the
   default unless configured otherwise.
6. Validate definitions and partition mappings before launch.
7. Review Kubernetes inheritance, owner references, replicas, and custom CA
   settings when deploying through Helm.

## Behavior that can surprise callers

- Partial run config is merged with job-level defaults for omitted fields.
- Date-like quoted YAML values remain strings.
- BigQuery, Snowflake, and DuckDB IO managers skip empty DataFrame writes and
  log a warning.
- A missing result from a blocking asset check warns and lets downstream
  assets proceed; an emitted failed check still fails the step.
- Wiping an asset or partition also clears its asset-check history.
- Backfills retry transient daemon failures, and failed asset backfills cancel
  in-progress runs before termination.
- Schedule, sensor, and asset-daemon ticks dispatch round-robin across code
  locations.

## Focused lookup

For API migration details, use
[upgrades-and-api.md](references/upgrades-and-api.md). For definition and
automation behavior, use
[assets-and-automation.md](references/assets-and-automation.md). Keep
integration-specific defaults and extension points paired with their package
guidance in [data-integrations.md](references/data-integrations.md) and
[platform-integrations.md](references/platform-integrations.md).
