# Data and analytics integrations

## dbt

In 1.11.0, `dagster-dbt` supports dbt Core 1.10 and has preview support for
the dbt Fusion CLI without application changes. Fusion is preferred when both
Fusion and dbt Core are installed, while `dbt-core` remains a dependency for
dbt Cloud. Configure component execution with `DbtProjectComponent.cli_args`.
Set `DAGSTER_DBT_CLOUD_POLL_INTERVAL` and
`DAGSTER_DBT_CLOUD_POLL_TIMEOUT` for dbt Cloud polling.

`DbtProjectComponent` exposes `get_asset_spec` and `get_asset_check_spec` as
extension points. In 1.12.0 it adds `op_config_schema` for runtime config.
`DbtProject` and its Component expose `prepare_project_cli_args` for
manifest-generation arguments. `dagster-dbt` supports dbt Core 1.11 and
prefers an installed `dbt-core` for manifest parsing.

`DbtCloudComponent` loads dbt Cloud projects as assets and can add a polling
sensor for Cloud jobs. `dbt_cloud_assets` accepts `partitions_def`.

In 1.13.0, `DagsterDbtTranslatorSettings.enable_source_metadata` defaults to
`True`, so dbt source table names remap upstream asset keys by default.
`DbtCloudComponent` adds custom `translation`; it and the workspace accept a
configurable job pool. `DbtProjectComponent.include_metadata` accepts
`"insights"` in YAML for Dagster+ Insights. dbt views can become virtual
assets through `enable_dbt_views_as_virtual_assets`.

## Airbyte

`AirbyteWorkspaceComponent`, renamed from
`AirbyteCloudWorkspaceComponent`, and `@airbyte_assets` support Airbyte OSS
and Enterprise as of 1.11.0. The Component exposes overridable
`get_asset_spec` and `execute`, and no longer reserves the old `io_manager` or
`airbyte` resource keys.

In 1.12.0, `AirbyteWorkspace` adds `poll_previous_running_sync`,
`max_items_per_page`, `poll_interval`, `poll_timeout`, and
`cancel_on_termination`.

See the upgrade reference for removed `AirbyteState` and legacy policy
arguments. Airbyte's state-backed Component defaults to local filesystem state
in 1.13.0 unless configured otherwise.

## Fivetran

`FivetranWorkspace` is GA in 1.11.0. Its Component exposes overridable
`get_asset_spec` and `execute` and no longer reserves the former `io_manager`
or `fivetran` resource keys.

In 1.12.0, the integration adds a polling sensor that represents externally
triggered syncs as materializations. `FivetranWorkspace` supports
`request_backoff_factor`, `retry_on_reschedule`, and resync operations for
request failures or quota-rescheduled syncs.

The Fivetran Component can opt into column-level metadata with
`fetch_column_metadata` in 1.13.0. Its state-backed storage defaults to local
filesystem state unless overridden.

## Sling and dlt

Sling and dlt Components expose overridable `get_asset_spec` (1.11.0).
`dagster-sling` and `dagster-dlt` support concurrency pools.

`DltLoadCollectionComponent` accepts `partitions_def` and `backfill_policy`
in 1.13.0.

## Power BI, Tableau, and other BI Components

Power BI Components expose overridable `get_asset_spec` (1.11.0).

`TableauComponent` can create materializable embedded and published
datasource assets with `enable_embedded_datasource_refresh` and
`enable_published_datsource_refresh` (spelled exactly as the API exposes it).
Filter with `workbook_selector` and `project_selector` (1.12.0).

New 1.12.0 Components cover Sigma, Looker, Tableau, Omni, Census, and
Polytomic. Use the asset-spec loaders described in the upgrade reference for
the newer Looker, Power BI, and Sigma definition-loading API.

## Warehouse IO managers

The `dagster-snowflake-polars` package provides
`SnowflakePolarsIOManager` (1.11.0).

`BigQueryIOManager.write_mode` accepts `truncate`, `replace`, or `append`
(1.12.0). From 1.13.0, BigQuery, Snowflake, and DuckDB IO managers skip empty
DataFrame writes and warn.

The `dagster-deltalake` and `dagster-deltalake-polars` integrations require
`deltalake>=1.0.0` as of 1.11.0, with no user-facing API change.

## Microsoft Teams

`dagster-msteams` can send Adaptive Card messages to PowerAutomate flows
(1.10.0).
