# Cloud and secrets

## Cloud command migration

Since 2.0.0, use explicit commands:

```sh
k6 cloud login
k6 cloud run script.js
k6 cloud upload script.js
```

The top-level `k6 login`, positional `k6 cloud script.js`, and
`--upload-only` are removed. InfluxDB credentials now come from
`K6_INFLUXDB_*`, not `k6 login influxdb`.

Move script configuration from `options.ext.loadimpact` to `options.cloud`:

```javascript
export const options = {
  cloud: { projectID: 12345, name: 'My Test' },
};
```

## Stacks and project selection

Since 1.6.0, Cloud login can save a default stack. Commands can use its slug or
ID to resolve the default project; override it with `K6_CLOUD_STACK_ID` or
`options.cloud.stackID`. Stack information was planned to become mandatory in
v2.

```sh
k6 cloud login --token "$MY_TOKEN" --stack my-stack-slug
K6_CLOUD_STACK_ID=12345 k6 cloud run script.js
```

Since 2.0.0, list projects as a table or JSON:

```sh
k6 cloud project list
k6 cloud project list --format=json
```

Since 2.1.0, list load tests with `k6 cloud test list`. Project resolution
uses `--project-id`, then `K6_CLOUD_PROJECT_ID` or cloud `projectID`, then the
configured stack's default project. Output is a table unless `--json` is used:

```sh
k6 cloud test list --project-id 12345
k6 cloud test list --json
```

## Local Cloud execution

The experimental provisioning flow in 1.0.0-rc2 accepted
`K6_BINARY_PROVISIONING=true` for `k6 cloud` and
`k6 cloud --local-execution`. It was restricted to supported extensions,
did not work with `k6 run`, and local execution still required a Cloud token.
Automatic extension resolution superseded that flow and the variable was
removed in v2.

Since 2.0.0, `k6 cloud run --local-execution` automatically enables the
built-in Cloud secret source, allowing `k6/secrets` to retrieve Cloud secrets
without `--secret-source=cloud`. Pass `--no-cloud-secrets` to opt out.

Since 1.8.0, local Cloud execution honors `K6_CLOUD_PUSH_REF_ID`; supplying an
existing run ID reuses that Cloud test run:

```sh
K6_CLOUD_PUSH_REF_ID="$RUN_ID" k6 cloud run --local-execution script.js
```

## Exit status and logs

Since 2.0.0, a Cloud run aborted by the system, a limit, a script error, the
user, or a timeout exits `97`. Success is `0`, and a threshold abort is `99`.
CI must treat `97` as failure.

Since 2.1.0, browser API failures in Cloud Logs carry `module=browser`.

## Secret sources

The asynchronous `k6/secrets` API arrived in 1.0.0-rc1. It reads configured
secret sources and redacts retrieved values from logs, including inside logged
responses. Built-in sources read a key-value file or CLI flags; extensions can
provide more:

```javascript
import secrets from 'k6/secrets';

export default async function () {
  console.log(await secrets.get('cool'));
}
```

```sh
k6 run --secret-source=mock=cool="not cool secret" script.js
```

Redaction includes `float32` and `float64` values since 1.0.0-rc2.

Since 1.5.0, secret management can fetch from HTTP URLs. That release supplied
only a mock implementation, not a production-ready external-secret-manager
integration.

Since 1.7.0, `K6_SECRET_SOURCE` is equivalent to `--secret-source` and accepts
the same syntax:

```sh
K6_SECRET_SOURCE='mock=cool="not cool secret"' k6 run script.js
```
