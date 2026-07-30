# Assets and automation

## Values, dependencies, and definitions

Since 1.11.0, `MaterializeResult(value=...)` invokes the asset IO manager and
may be typed as `MaterializeResult[T]`. `AssetExecutionContext.load_asset_value`
loads another asset dynamically through its IO manager rather than requiring
the value as a function parameter.

```python
import dagster as dg

@dg.asset
def upstream() -> dg.MaterializeResult[int]:
    return dg.MaterializeResult(value=42)

@dg.asset(deps=[upstream])
def downstream(context: dg.AssetExecutionContext):
    return context.load_asset_value(dg.AssetKey("upstream"))
```

`Definitions` and `AssetsDefinition` reject distinct `AssetSpec` objects that
share an asset key. Definition validation also rejects invalid partition
mappings, including time-partitioned dependencies whose time zones differ.

Configurable resource fields may use union annotations such as `Foo | Bar`.
To hide a resource parameter in the UI, define its Pydantic field with
`json_schema_extra={"dagster__is_secret": True}` (1.12.0).

Partial run config now fills omitted sections from job-level config defaults
(1.13.0). Op and asset inputs accept `typing.Mapping` and `typing.Sequence`,
and `TableMetadataSet.storage_kind` records the backing system, such as
Snowflake, Databricks, or BigQuery.

## Virtual assets and metadata

The preview `is_virtual` parameter on `@asset` and `AssetSpec` represents
views or derived tables that change with upstream data without explicit
materialization (1.13.0). Virtual assets participate in staleness
calculation, execution planning, and declarative automation. dbt views can be
marked automatically with `enable_dbt_views_as_virtual_assets`.

An asset may carry up to ten kind annotations as of 1.12.0, up from three.
`define_asset_job` and `build_schedule_from_partitioned_job` accept `owners`
in 1.13.0, and asset-job owners are validated during definition loading.
Team-owner strings on jobs, schedules, and sensors may contain special
characters.

## Checks, hooks, and retries

Ops may yield `AssetCheckEvaluation`, and `@asset` accepts `hooks` for success
and failure callbacks (1.11.0). Re-execution can target only the failed assets
inside a failed multi-asset step rather than rerunning every asset in it.

Since 1.12.0, `@asset_check` and `AssetCheckSpec` accept `partitions_def`.
That definition must match the target asset's partition definition.

Asset-check edge behavior changed in 1.13.0:

- Assets whose names contain dots may be check targets.
- If a blocking check emits no result, downstream assets proceed with a
  warning. An emitted failed check still fails the step.
- Wiping an asset or selected partitions clears their asset-check history.

## Partitions, backfills, and schedules

Time-window partition definitions accept `exclusions` for custom calendars
(1.11.0). `OpExecutionContext`, `AssetExecutionContext`, and
`AssetCheckExecutionContext` expose `multi_partition_key` for
multi-partition runs (1.12.0).

`BackfillPolicy` is GA in 1.11.0. Backfill submission uses four daemon worker
threads by default, asset backfills can receive run config, and a failed
backfill cancels in-progress runs before terminating. A schedule
`RunRequest` may choose a subset of asset checks.

In 1.13.0, job backfills retry transient daemon failures. The environment
variable `DAGSTER_MAX_ASSET_BACKFILL_RETRIES` was renamed to
`DAGSTER_MAX_BACKFILL_RETRIES`; the old name remains a fallback.

## Step dependency execution

Every executor understands
`step_dependency_config.require_upstream_step_success` (1.11.0). Set it to
`false` when a downstream step may start once the required upstream outputs
exist, even while their producing multi-asset step is still running:

```json
{"step_dependency_config": {"require_upstream_step_success": false}}
```

## Automation conditions

`AutomationCondition.data_version_changed()` can trigger when an asset's data
version changes (1.10.0).

The run-tag-aware conditions added in 1.11.0 inspect new events since the
previous tick:

- `all_new_updates_have_run_tags()` and
  `any_new_update_has_run_tags()` consider every new materialization, not just
  the latest run.
- `all_new_executed_with_tags()` filters newly executed partitions by tags.

Freshness-aware `freshness_passed()`, `freshness_warned()`, and
`freshness_failed()` branch on the latest policy evaluation (1.12.0).

## Selection expressions and groups

Since 1.11.0, selection expressions combine lineage traversal, attribute
filters, and Boolean logic. The syntax is shared by Components YAML, Asset
Catalog, saved selections, alerts, and insights; Gantt views have analogous
op selections.

Partition-definition type filters are supported, for example
`partitions:"static"` (1.12.0).

The 1.13.0 syntax adds `sensor:`, `schedule:`, `job:`, and
`automation_type:` attributes plus `is:` type filters:

```text
sensor:daily_refresh
automation_type:schedule
is:materializable
```

Schedule and sensor selectors include assets in targeted jobs as well as
assets directly selected by the instigator. Group names may contain `/`,
render as a hierarchy, and support wildcards such as
`group:"marketing/*"`.

## GraphQL and API selections

`DagsterGraphQLClient.submit_job_execution` accepts `asset_selection`
(1.11.0). The `logsForRun` and `eventConnection` resolvers return at most
1,000 events by default; follow the returned cursor for the rest.

In 1.13.0, `DagsterGraphQLClient` accepts `path_prefix` for a webserver
mounted below the URL root. GraphQL `Run` adds an optional selection `limit`
and `assetSelectionCount` / `assetCheckSelectionCount`, allowing a bounded
preview while preserving total counts.

## YAML and failure context

Quoted date-like YAML strings such as `"2021-10-30"` remain strings rather
than becoming datetimes (1.13.0). If a run fails because a step failed, the
originating step error is available on the run-failure sensor context.
