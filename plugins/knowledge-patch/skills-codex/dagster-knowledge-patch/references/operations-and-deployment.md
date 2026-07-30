# Operations and deployment

## Custom executors and resource initialization

Dedicated-process executors can recover from resource initialization failures
through step retries (1.12-upgrade). A custom `Executor` must emit and
register a failure-or-retry event for each failure. Without it, the run can
remain in `started`:

```python
if event.is_resource_init_failure:
    failure_or_retry_event = self.get_failure_or_retry_event_after_error(
        step_context,
        event.engine_event_data.error,
        active_execution.get_known_state(),
    )
    yield failure_or_retry_event
    active_execution.handle_event(failure_or_retry_event)
```

## Event and proxy limits

Since 1.11.0, event error messages or stack traces larger than 500 KB are
truncated. Override the threshold with
`DAGSTER_EVENT_ERROR_FIELD_SIZE_LIMIT`.

Kubernetes executor `enable_owner_references` ties step jobs and pods to the
run pod for garbage collection. `DAGSTER_GRPC_PROXY_HEARTBEAT_TTL_SECONDS`
sets proxy gRPC heartbeat TTL; the default is 30 seconds.

## Code locations, pools, and daemons

Runs with a remote job origin automatically receive the
`dagster/code_location` tag (1.12.0), useful for filters and concurrency
controls. Helm configuration supports a `concurrency` setting for pools.

Schedule, sensor, and asset-daemon ticks dispatch instigators round-robin
across code locations in 1.13.0. Job backfills retry transient daemon
failures.

## Kubernetes and Helm

The 1.12.0 Helm chart supports image digests. Dagster and Dagster+ agent
charts accept `k8sApiCaBundlePath` for a custom Kubernetes API CA.
Code-location Services accept arbitrary Kubernetes Service overrides through
`service_spec_config`, and the supported Kubernetes dependency range includes
35.x.

With `includeConfigInLaunchedRuns.enabled` in 1.13.0, launched run pods inherit
`nodeSelector`, `tolerations`, and `podSecurityContext` from the user
deployment. User-code deployments accept `replicaCount`; replicas share a
stable gRPC server ID. `code_server.*` metrics identify the responding process
through `server_instance_id`.

For sovereign Azure, ADLS2 and Blob Storage utilities, resources, Components,
and compute logging accept `endpoint_suffix`. The corresponding compute-log
Helm field is `endpointSuffix`.

## ECS behavior

Jobs and Launchpad runs using `EcsRunLauncher` can use the
`ecs/container_overrides` tag for settings such as GPU requirements (1.12.0).

`EcsUserCodeLauncher.repository_credentials` can configure ECR credentials at
agent or deployment scope in 1.13.0, not only per code location.

ECS stops caused by `InsufficientFreeAddressesInSubnet` or
`Task provisioning failed` are classified as transient in 1.13.0, so the
affected run is retried instead of marked permanently failed.

## Authentication

`DatabricksClientResource.credentials_strategy` accepts the Databricks SDK
`CredentialsStrategy` protocol for custom or federated authentication
(1.13.0).

PostgreSQL accepts `auth_provider="azure_wif"`, `"gcp_wif"`, or `"aws_wif"`,
with corresponding optional extras. Helm exposes
`global.postgresqlAuthWifEnabled`.

## Storage and IO behavior

The SQLite event-log `busy_timeout` default rose from 5 to 30 seconds in
1.13.0. `PickledObjectS3IOManager` uses an empty key prefix when none is
provided.

BigQuery, Snowflake, and DuckDB IO managers skip empty DataFrame writes and
log a warning rather than creating a table from degenerate inferred types.

## Pipes controls

The preview `PipesCompositeMessageReader` handles multiple concurrent message
streams within one Pipes session (1.13.0).
`PipesK8sClient.run(delete_pod_on_completion=False)` preserves its pod after
completion. `PipesEMRServerlessClient.dashboard_refresh_interval` controls
Spark dashboard refresh; its longer default keeps UI URLs valid during runs.

## Runtime and notebook compatibility

The `dagstermill` package requires `papermill>=2.0.0` in 1.13.0 and raises
the default Jupyter kernel startup timeout from 60 to 120 seconds.
`dagster-airlift` supports Python 3.12, 3.13, and 3.14.

## Administrative APIs

Dagster+ SCIM Groups queries support the `members.value eq` filter (1.13.0).
