# Integrations

Use this reference when configuring integration packages and Components. Check
the installed integration package independently from core Dagster.

## dbt

### Execution engines and project controls

In 1.11.0, `dagster-dbt` supports dbt Core 1.10 and preview use of the dbt
Fusion CLI without code changes. `dbt-core` remains required for dbt Cloud;
Fusion is preferred when both engines are installed. Configure component
execution with `DbtProjectComponent.cli_args`.

In 1.12.0, dbt Core 1.11 is supported. `DbtProject` and
`DbtProjectComponent` expose `prepare_project_cli_args` for manifest-generation
arguments, and an installed `dbt-core` is preferred for manifest parsing.
`DbtProjectComponent.op_config_schema` customizes runtime configuration.

`DAGSTER_DBT_CLOUD_POLL_INTERVAL` and `DAGSTER_DBT_CLOUD_POLL_TIMEOUT` control
dbt Cloud polling (1.11.0).

### dbt Cloud and asset metadata

`DbtCloudComponent` loads dbt Cloud projects as assets and can add a polling
sensor for Cloud job runs (1.12.0). The `dbt_cloud_assets` decorator accepts
`partitions_def` for partitioned assets.

As of 1.13.0, `DbtCloudComponent` supports custom `translation`; it and the
workspace accept a configurable job pool. `DbtProjectComponent.include_metadata`
accepts `"insights"` in YAML for Dagster+ Insights tracking.

`DagsterDbtTranslatorSettings.enable_source_metadata` defaults to `True` in
1.13.0, so dbt source table names remap upstream asset keys by default. Audit
keys when upgrading. dbt views can opt into virtual assets automatically.

The dbt Cloud integration was beta in 1.11.0. A preview
`SnowflakeDbtProjectComponent` for native Snowflake dbt orchestration arrived
in 1.13.0.

## Airbyte

`AirbyteWorkspaceComponent`, renamed from `AirbyteCloudWorkspaceComponent`,
and `@airbyte_assets` support Airbyte OSS and Enterprise as of 1.11.0.

`AirbyteWorkspace` added these controls in 1.12.0:

- `poll_previous_running_sync`
- `max_items_per_page`
- `poll_interval`
- `poll_timeout`
- `cancel_on_termination`

Airbyte Components use the state-backed discovery model. Review the local
filesystem default described in the Components reference.

## Fivetran

`FivetranWorkspace` is GA as of 1.11.0. In 1.12.0, the integration added a
polling sensor that converts externally triggered syncs into materializations.
Workspace controls include `request_backoff_factor`, `retry_on_reschedule`,
and resync operations for request failures and quota-rescheduled syncs.

The Fivetran Component can fetch column-level metadata with
`fetch_column_metadata` as of 1.13.0.

## Databricks

The 1.11.0 integration added `PipesDatabricksServerlessClient`, a preview
`DatabricksAssetBundleComponent`, Spark Python and Python Wheel serverless
tasks, and `notebook_task` support in `PipesDatabricksClient`.

`DatabricksWorkspaceComponent` discovers jobs as assets and cancels a
Databricks job when its Dagster run is terminated (1.12.0).
`DatabricksAssetBundleComponent` is subsettable by task at the job level and
uses the Databricks CLI to resolve bundle-variable references.

`DatabricksClientResource.credentials_strategy` accepts the Databricks SDK
`CredentialsStrategy` protocol for federated or custom authentication as of
1.13.0.

## BI and analytics systems

### Tableau

`TableauComponent` can create materializable embedded and published datasource
assets (1.12.0). Enable refresh with
`enable_embedded_datasource_refresh` and
`enable_published_datsource_refresh`—note the spelling of `datsource` in the
published option. Filter with `workbook_selector` and `project_selector`.

### Additional Components

Components added in 1.12.0 cover Sigma, Looker, Tableau, Omni, Census, and
Polytomic. Declarative resource Components were also added for AWS, Azure, and
GCP.

The 1.13.0 preview integrations include `SodaScanComponent` for Soda Core.
Looker, Power BI, and Sigma definition-loading removals are covered in the
upgrade reference.

## Data movement and orchestration Components

The Airflow Component was beta in 1.11.0. Power BI, Sling, dlt, Airbyte, and
Fivetran expose Component extension points described in the Components
reference.

`DltLoadCollectionComponent` accepts `partitions_def` and `backfill_policy`
as of 1.13.0.

## Database and table IO

The `dagster-snowflake-polars` package introduced in 1.11.0 provides
`SnowflakePolarsIOManager`. The Apache Iceberg IO manager was preview at that
time.

`BigQueryIOManager.write_mode` accepts `truncate`, `replace`, or `append` as
of 1.12.0.

The new 1.13.0 packages `dagster-clickhouse`,
`dagster-clickhouse-pandas`, and `dagster-clickhouse-polars` provide native
resources, IO managers, and `dg` Components.

As of 1.13.0, BigQuery, Snowflake, and DuckDB IO managers skip empty DataFrame
writes and log a warning instead of creating a table from degenerate inferred
types.

## Pipes and cloud execution

Dagster Pipes added `PipesAzureMLClient` and Azure Blob Storage support in
1.12.0.

Preview `PipesCompositeMessageReader` handles concurrent message streams in a
single Pipes session as of 1.13.0.
`PipesK8sClient.run(delete_pod_on_completion=False)` retains the pod.
`PipesEMRServerlessClient.dashboard_refresh_interval` controls Spark-dashboard
refresh and has a longer default so UI URLs remain valid while runs execute.

## Microsoft Teams and PowerAutomate

`dagster-msteams` can send Adaptive Card messages to PowerAutomate flows as of
1.10.0.
