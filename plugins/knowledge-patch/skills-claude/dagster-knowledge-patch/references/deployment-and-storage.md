# Deployment and Storage

Use this reference for Helm, Kubernetes, ECS, cloud authentication, databases,
storage defaults, and deployment-specific runtime behavior.

## Helm and Kubernetes

The Dagster Helm chart supports image digests and a `concurrency` pool setting
as of 1.12.0. Dagster and Dagster+ agent charts accept
`k8sApiCaBundlePath` for a custom Kubernetes API CA. Code-location Services
accept arbitrary Kubernetes Service overrides through `service_spec_config`.
The supported Kubernetes dependency range includes 35.x.

With `includeConfigInLaunchedRuns.enabled`, launched run pods inherit
`nodeSelector`, `tolerations`, and `podSecurityContext` from the user
deployment as of 1.13.0.

User-code deployments accept `replicaCount` (1.13.0). Replicas share a stable
gRPC server ID, while `code_server.*` metrics identify the responding process
through `server_instance_id`.

## ECS

Jobs and launchpad runs using `EcsRunLauncher` can set the
`ecs/container_overrides` tag for settings such as GPU requirements (1.12.0).

`EcsUserCodeLauncher.repository_credentials` can configure ECR credentials at
agent or deployment scope as of 1.13.0, rather than only at each code location.

ECS capacity stops for `InsufficientFreeAddressesInSubnet` and
`Task provisioning failed` are transient in 1.13.0 and trigger run retries.

## Federated database authentication

PostgreSQL accepts `auth_provider="azure_wif"`, `"gcp_wif"`, or `"aws_wif"`
as of 1.13.0:

- `azure_wif`
- `gcp_wif`
- `aws_wif`

Install the corresponding optional extra. For Helm deployments, enable
`global.postgresqlAuthWifEnabled`.

## Sovereign Azure endpoints

ADLS2 and Blob Storage utilities, resources, Components, and compute logging
accept `endpoint_suffix` for sovereign clouds (1.13.0). The Helm compute-log
setting uses the camel-cased spelling `endpointSuffix`.

## Database migrations and development pools

MySQL users must run `dagster instance migrate` for the 1.12.0 `LongText`
migrations covering bulk-action bodies and cached asset-status data.

`dagster-postgres` no longer installs `psycopg2-binary` transitively as of
1.12.0. Declare the driver directly when the project uses it.

`dg dev` and `dagster dev` accept development database-pool controls including
`--db-pool-recycle` and `--db-pool-pre-ping` (1.12.0).

## Storage behavior

The SQLite event-log `busy_timeout` defaults to 30 seconds rather than 5 as of
1.13.0.

`PickledObjectS3IOManager` uses an empty key prefix when none is supplied
(1.13.0).

BigQuery, Snowflake, and DuckDB IO managers skip writes for empty DataFrames
and warn instead of inferring and creating a degenerate table (1.13.0).

## Deployment runtime and cleanup

Set the Kubernetes executor's `enable_owner_references` option to bind step
jobs and pods to the run pod for garbage collection (1.11.0).

Event error messages or stack traces over 500 KB are truncated by default.
Set `DAGSTER_EVENT_ERROR_FIELD_SIZE_LIMIT` to override the limit (1.11.0).

The proxy gRPC heartbeat TTL defaults to 30 seconds and can be changed with
`DAGSTER_GRPC_PROXY_HEARTBEAT_TTL_SECONDS` (1.11.0).

## Dagster+ SCIM

SCIM Groups queries support the `members.value eq` filter as of 1.13.0.
