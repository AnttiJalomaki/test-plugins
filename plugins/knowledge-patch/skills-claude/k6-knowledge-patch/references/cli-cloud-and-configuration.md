# CLI, Cloud, and Configuration

## Container execution

### Account for the numeric image user (since 1.1.0)

The container image selects numeric UID `12345` instead of the named `k6`
user. Kubernetes pods therefore do not need a separate `runAsUser` setting
just to match the image's selected user.

### Pin the intended Docker major line (since 2.0.0)

Prereleases and maintenance releases from older major lines do not update the
Docker `:latest` tag or the GitHub latest-release marker. Use a floating major
tag such as `grafana/k6:v1` when deployment should track a chosen major line.

## Summary modes

### Disable the end-of-test summary (since 1.3.0)

`--no-summary` and `K6_NO_SUMMARY` are deprecated. Select the `disabled`
summary mode instead:

```sh
k6 run --summary-mode=disabled script.js
K6_SUMMARY_MODE=disabled k6 run script.js
```

### Leave legacy mode (since 1.3.0)

The `legacy` summary mode is deprecated and was planned for removal in v2.
Select `compact` or `full` instead.

## Default Cloud stack and project resolution

### Save a default stack (since 1.6.0)

Cloud login can save a default stack by slug or ID. Cloud commands use that
stack to find the default project. Override the stack for a run with
`K6_CLOUD_STACK_ID` or `options.cloud.stackID`. Stack information was announced
as mandatory for v2.

```sh
k6 cloud login --token "$MY_TOKEN" --stack my-stack-slug
K6_CLOUD_STACK_ID=12345 k6 cloud run script.js
```

### Move script configuration to `options.cloud` (since 2.0.0)

`options.ext.loadimpact` is rejected. Move its fields under `options.cloud`:

```javascript
export const options = {
  cloud: { projectID: 12345, name: 'My Test' },
};
```

## Cloud command migration

### Use explicit Cloud commands (since 2.0.0)

The top-level `k6 login`, positional `k6 cloud script.js`, and `--upload-only`
interfaces were removed. Use:

```sh
k6 cloud login
k6 cloud run script.js
k6 cloud upload script.js
```

Supply InfluxDB credentials through `K6_INFLUXDB_*` environment variables;
`k6 login influxdb` is no longer available.

### List projects (since 2.0.0)

List Grafana Cloud k6 projects as a table or JSON:

```sh
k6 cloud project list
k6 cloud project list --format=json
```

### List tests and resolve the project (since 2.1.0)

`k6 cloud test list` lists load tests in a Grafana Cloud k6 project. Project
resolution checks, in order:

1. `--project-id`.
2. `K6_CLOUD_PROJECT_ID` or cloud `projectID` configuration.
3. The configured stack's default project.

The default output is a table; use `--json` for structured output.

```sh
k6 cloud test list --project-id 12345
k6 cloud test list --json
```

## Local Cloud execution

### Load Cloud secrets automatically (since 2.0.0)

`k6 cloud run --local-execution` enables the built-in Cloud secret source, so
`k6/secrets` can retrieve Grafana Cloud secrets without
`--secret-source=cloud`. Pass `--no-cloud-secrets` to opt out.

### Reuse an existing Cloud run (since 1.8.0)

Set `K6_CLOUD_PUSH_REF_ID` during local execution to reuse that Cloud test run
instead of creating a new one:

```sh
K6_CLOUD_PUSH_REF_ID="$RUN_ID" k6 cloud run --local-execution script.js
```

## Exit statuses

### Treat a Cloud abort as failure (since 2.0.0)

A Cloud run aborted by the system, a limit, a script error, the user, or a
timeout exits with status `97` instead of `0`. Successful runs remain `0`, and
threshold aborts remain `99`. CI should treat `97` as failure.

## Configuration paths and services

### Move the legacy configuration file (since 2.0.0)

k6 no longer reads or migrates
`{USER_CONFIG_DIR}/loadimpact/config.json`. Move it to
`{USER_CONFIG_DIR}/k6/config.json`, or regenerate configuration with
`k6 cloud login`.

### Opt in to the HTTP API (since 2.0.0)

The HTTP API does not listen on `localhost:6565` by default. Enable it with
`--address` or `K6_ADDRESS`:

```sh
k6 run --address=localhost:6565 script.js
```

## Feature flags

### Enable and inspect experimental features (since 2.1.0)

Enable features for `k6 run` and `k6 cloud run` with repeated or
comma-separated `--features` flags, `K6_FEATURES`, or the `features` key in
`config.json`. Inspect available flags and their lifecycle with `k6 features`
or `k6 features --json`.

Enabled features are added to metric tags and preserved in archives and Cloud
workers. See [Native histograms](outputs-and-observability.md#native-histograms)
for the first feature's metric behavior.

```sh
k6 run --features native-histograms script.js
K6_FEATURES=native-histograms k6 run script.js
```

## Execution overrides

### Understand `--vus` with configured scenarios (since 2.1.0)

When the script defines scenarios, `k6 run script.js --vus N` warns and
replaces them with a `shared-iterations` scenario containing `N` VUs and `N`
iterations. The flag is no longer silently ignored in that situation.
