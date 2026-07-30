# Platform and compute integrations

## Integration maturity and extension points

In 1.11.0, the Airflow Component and dbt Cloud integration are beta, and the
Apache Iceberg IO manager is preview. Airbyte, Fivetran, Power BI, Sling, and
dlt Components expose overridable `get_asset_spec`; Airbyte and Fivetran also
expose `execute`.

## Databricks

`PipesDatabricksServerlessClient` arrived in 1.11.0 along with a preview
`DatabricksAssetBundleComponent`. Databricks supports Spark Python and Python
Wheel serverless tasks, and `PipesDatabricksClient` supports `notebook_task`.

In 1.12.0, `DatabricksWorkspaceComponent` discovers Databricks jobs as assets
and cancels them when their Dagster run is terminated.
`DatabricksAssetBundleComponent` is subsettable by task at the job level and
uses the Databricks CLI to resolve bundle variable references.

Federated and custom authentication through `CredentialsStrategy` is
described in the operations reference.

## Cloud resource Components and Pipes

Declarative AWS, Azure, and GCP resource Components were added in 1.12.0.
Dagster Pipes adds `PipesAzureMLClient` and Azure Blob Storage support.

Sovereign Azure endpoints, federated PostgreSQL authentication, ECR repository
credentials, and newer Pipes runtime controls are covered in the operations
reference.

## New packages and Components

The 1.13.0 packages `dagster-clickhouse`,
`dagster-clickhouse-pandas`, and `dagster-clickhouse-polars` provide native
resources, IO managers, and `dg` Components.

Preview integrations add `SodaScanComponent` for Soda Core and
`SnowflakeDbtProjectComponent` for native dbt orchestration on Snowflake.
