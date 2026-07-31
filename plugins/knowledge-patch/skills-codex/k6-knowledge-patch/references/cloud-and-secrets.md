# Cloud and Secrets

## Select stacks and projects

### Save a default Cloud stack

Cloud commands can save a default stack during login and use its slug or ID to
resolve the default project (since 1.6.0):

```sh
k6 cloud login --token "$MY_TOKEN" --stack my-stack-slug
K6_CLOUD_STACK_ID=12345 k6 cloud run script.js
```

Override the saved stack with `K6_CLOUD_STACK_ID` or
`options.cloud.stackID`. Stack information was announced as mandatory for v2,
so configure it explicitly in durable automation.

### List projects and tests

k6 v2 can list Cloud projects in a table or as JSON (2.0.0):

```sh
k6 cloud project list
k6 cloud project list --format=json
```

`k6 cloud test list` lists tests in a project (since 2.1.0):

```sh
k6 cloud test list --project-id 12345
k6 cloud test list --json
```

Project resolution uses this precedence:

1. `--project-id`
2. `K6_CLOUD_PROJECT_ID` or cloud `projectID`
3. the configured stack's default project

The default output is a table; `--json` selects machine-readable output.

## Migrate Cloud commands and configuration

k6 v2 removed the top-level `k6 login`, positional `k6 cloud script.js`, and
`--upload-only` forms (2.0.0). Use explicit commands:

```sh
k6 cloud login
k6 cloud run script.js
k6 cloud upload script.js
```

Supply InfluxDB credentials through `K6_INFLUXDB_*` variables; `k6 login
influxdb` is gone. Move fields from the removed `options.ext.loadimpact`
namespace to `options.cloud`:

```javascript
export const options = {
  cloud: { projectID: 12345, name: 'My Test' },
};
```

The legacy `{USER_CONFIG_DIR}/loadimpact/config.json` path is not migrated or
read by v2. Move the file to `{USER_CONFIG_DIR}/k6/config.json` or regenerate
it with `k6 cloud login` (2.0.0).

## Handle Cloud exit status in CI

A Cloud run aborted by the system, a limit, a script error, the user, or a
timeout exits with status `97` in v2 (2.0.0). Successful runs remain `0`, and
threshold aborts remain `99`. CI must treat `97` as failure rather than relying
on the earlier zero status.

## Reuse a Cloud run for local execution

`k6 cloud run --local-execution` honors `K6_CLOUD_PUSH_REF_ID` (since 1.8.0).
Set an existing run ID to reuse that Cloud test run instead of creating one:

```sh
K6_CLOUD_PUSH_REF_ID="$RUN_ID" k6 cloud run --local-execution script.js
```

## Use Cloud secrets during local execution

In v2, `k6 cloud run --local-execution` automatically enables the built-in
Cloud secret source (2.0.0). Calls through `k6/secrets` can retrieve Grafana
Cloud secrets without `--secret-source=cloud`. Pass `--no-cloud-secrets` when
the local run must not access them.

## Configure other secret sources

### Supply source configuration through the environment

`K6_SECRET_SOURCE` accepts the same syntax as `--secret-source` (since 1.7.0):

```sh
K6_SECRET_SOURCE='mock=cool="not cool secret"' k6 run script.js
```

Use the environment form when a runner cannot safely or conveniently construct
the repeated CLI option.

### Fetch secrets from URLs

Secret management can fetch values from HTTP endpoints (since 1.5.0), allowing
a custom service to provide secrets. That release provides only a mock
implementation, not a production-ready external-secret-manager integration.
Do not describe the mock URL source as a hardened production connector.
