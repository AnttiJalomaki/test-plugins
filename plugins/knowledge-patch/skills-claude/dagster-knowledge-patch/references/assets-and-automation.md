# Assets and Automation

Use this reference for asset definitions, selection, partitions, freshness,
automation conditions, checks, ownership, and table metadata.

## Values and dependencies

Since 1.11.0, `MaterializeResult(value=...)` invokes the asset IO manager and
can be annotated as `MaterializeResult[T]`.
`AssetExecutionContext.load_asset_value` can load another asset dynamically
through its IO manager without making it a function parameter:

```python
import dagster as dg

@dg.asset
def upstream() -> dg.MaterializeResult[int]:
    return dg.MaterializeResult(value=42)

@dg.asset(deps=[upstream])
def downstream(context: dg.AssetExecutionContext):
    return context.load_asset_value(dg.AssetKey("upstream"))
```

Op and asset inputs accept `typing.Mapping` and `typing.Sequence` annotations
as of 1.13.0.

## Virtual assets and asset shape

Preview `is_virtual` on `@asset` and `AssetSpec` models views or derived tables
that follow upstream changes without explicit materialization (1.13.0).
Virtual assets participate in staleness calculation, execution planning, and
declarative automation. dbt views can be marked automatically with
`enable_dbt_views_as_virtual_assets`.

An asset may carry up to 10 kind annotations as of 1.12.0, increased from
three. Asset groups can contain `/` separators and render hierarchically as of
1.13.0; select descendants with patterns such as `group:"marketing/*"`.

## Selection expressions

Unified asset selections combine lineage traversal, attribute filters, and
Boolean logic (1.11.0). The same expression language is used in Components
YAML, the Asset Catalog, saved selections, alerts, and insights. Run Gantt
views have analogous op selections.

The 1.13.0 syntax adds `sensor:`, `schedule:`, `job:`, and
`automation_type:` attributes plus `is:` type filters:

```text
sensor:daily_refresh
schedule:hourly_load
job:warehouse_build
automation_type:schedule
is:materializable
```

Schedule and sensor selectors include assets in jobs targeted by the
instigator as well as directly selected assets. Select by partition-definition
type with expressions such as `partitions:"static"` (1.12.0).

## Partitions and calendars

Time-window partition definitions accept `exclusions` for custom calendars
since 1.11.0. Use them to omit holidays, maintenance dates, or specific times.

`OpExecutionContext`, `AssetExecutionContext`, and
`AssetCheckExecutionContext` expose `multi_partition_key` for multi-partition
runs as of 1.12.0.

## Automation conditions

Use `AutomationCondition.data_version_changed()` to trigger when an asset's
data version changes (1.10.0).

The run-tag-aware conditions introduced in 1.11.0 inspect materializations
since the previous tick rather than only the latest run:

- `AutomationCondition.all_new_updates_have_run_tags()`
- `any_new_update_has_run_tags()`
- `all_new_executed_with_tags()` for newly executed partitions

Freshness evaluation conditions added in 1.12.0 let automation branch on the
latest result:

- `AutomationCondition.freshness_passed()`
- `AutomationCondition.freshness_warned()`
- `AutomationCondition.freshness_failed()`

## Asset checks and hooks

Ops can yield `AssetCheckEvaluation`, and `@asset` accepts `hooks` for success
and failure callbacks as of 1.11.0. Re-execution can retry only failed assets
within a failed multi-asset step instead of rerunning every asset in the step.

`@asset_check` and `AssetCheckSpec` accept `partitions_def` as of 1.12.0, so a
check can target individual partitions. The supplied partition definition must
match the target asset's definition.

The following edge behavior applies as of 1.13.0:

- Checks can target assets whose names contain dots.
- If a blocking check emits no result, downstream assets proceed with a
  warning.
- If it emits a failed check, the step still fails.
- Wiping an asset or its partitions also clears associated check history.

A schedule's `RunRequest` can select a subset of asset checks (1.11.0).

## Freshness evaluation

Freshness policies supersede the older freshness-check builders. Automatic
evaluation is enabled by default; set `freshness.enabled: false` in
`dagster.yaml` only when evaluation should be disabled. See the upgrade
reference for removed imports and legacy parameters.

## Ownership and system tags

Runs with a remote job origin automatically receive the
`dagster/code_location` tag (1.12.0). Use it for filtering or concurrency
controls.

`define_asset_job` and `build_schedule_from_partitioned_job` accept `owners`
as of 1.13.0. Asset-job owners are validated when definitions load. Team-owner
strings for jobs, schedules, and sensors may contain special characters.

## Table metadata

`TableMetadataSet.storage_kind` records the backing table system, such as
Snowflake, Databricks, or BigQuery (1.13.0).
