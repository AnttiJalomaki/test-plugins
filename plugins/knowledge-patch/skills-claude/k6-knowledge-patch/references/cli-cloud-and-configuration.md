# CLI, Cloud, and Configuration

## CLI parsing and script scaffolding

Commas are literal parts of `--tag` values as of `1.0.0-rc1`; they no longer
separate multiple tag-value sets. Repeat the flag:

```sh
k6 run --tag 'label=a,b' --tag env=test script.js
```

`k6 new` accepts a Go-template file in `1.0.0-rc1`. Templates receive
`ScriptName` and `ProjectID`:

```sh
k6 new --template /path/to/my-template.js
```

In `2.1.0`, `k6 run script.js --vus N` warns and replaces configured scenarios
with one `shared-iterations` scenario containing `N` VUs and `N` iterations.
The flag is no longer silently ignored when the script defines scenarios.

## Configuration file location

`k6 cloud login` and the deprecated `k6 login` began creating
`{USER_CONFIG_DIR}/k6/config.json` in `1.0.0-rc1`. At that point, login
migrated `{USER_CONFIG_DIR}/loadimpact/config.json`, while `k6 run` preferred
the new path, fell back to the old one, and warned.

The fallback and migration were removed in `2.0.0`. Move an old file manually
or rerun `k6 cloud login`.

## Human end-of-test summaries

The redesigned summary in `1.0.0-rc1` introduced:

- `compact`, the default;
- `full`, with detailed metrics plus per-group and per-scenario results;
- `legacy`, which reproduced the old output.

The `handleSummary` input and `--summary-export` structure did not change in
that release. Threshold values became visible even when absent from
`summaryTrendStats` in `1.0.0-rc2`.

In `1.3.0`, use `disabled` instead of deprecated `--no-summary` or
`K6_NO_SUMMARY`:

```sh
k6 run --summary-mode=disabled script.js
K6_SUMMARY_MODE=disabled k6 run script.js
```

The `legacy` mode was also deprecated in `1.3.0`; choose `compact` or `full`.

## Cloud binary provisioning transition

In `1.0.0-rc2`, experimental Cloud binary provisioning required
`K6_BINARY_PROVISIONING=true`. It could build only a supported extension set
for `k6 cloud`; it did not apply to `k6 run`. Local execution still required a
Cloud token from the normal login flow:

```sh
K6_BINARY_PROVISIONING=true k6 cloud script.js
K6_BINARY_PROVISIONING=true k6 cloud --local-execution script.js
```

Automatic extension resolution replaced this opt-in flow later. Do not copy
the release-candidate commands into current v2 automation.

## Stacks, projects, and tests

Cloud commands can save a default stack during login as of `1.6.0`. The stack
slug or ID resolves its default project. Override the saved choice with
`K6_CLOUD_STACK_ID` or `options.cloud.stackID`; stack information was announced
as mandatory for v2.

```sh
k6 cloud login --token "$MY_TOKEN" --stack my-stack-slug
K6_CLOUD_STACK_ID=12345 k6 cloud run script.js
```

Project discovery became available in `2.0.0`:

```sh
k6 cloud project list
k6 cloud project list --format=json
```

`k6 cloud test list` was added in `2.1.0`. It resolves a project in this order:

1. `--project-id`;
2. `K6_CLOUD_PROJECT_ID` or cloud `projectID`;
3. the configured stack's default project.

Output is a table unless `--json` is supplied:

```sh
k6 cloud test list --project-id 12345
k6 cloud test list --json
```

## Cloud local execution

As of `2.0.0`, `k6 cloud run --local-execution` automatically enables the
built-in Cloud secret source, so it does not require
`--secret-source=cloud`. Pass `--no-cloud-secrets` to opt out.

As of `1.8.0`, `K6_CLOUD_PUSH_REF_ID` can point local execution at an existing
Cloud run instead of creating a new one:

```sh
K6_CLOUD_PUSH_REF_ID="$RUN_ID" k6 cloud run --local-execution script.js
```

## HTTP API and web dashboard

The k6 HTTP API became opt-in in `2.0.0`; use `--address` or `K6_ADDRESS`.

The web dashboard was built into the binary in `2.0.0`, replacing the need for
a separate xk6-dashboard extension:

```sh
k6 run --out=web-dashboard script.js
```

## Feature flags

`2.1.0` introduced a unified feature interface for `k6 run` and
`k6 cloud run`. Enable flags with repeated or comma-separated `--features`,
`K6_FEATURES`, or the `features` key in `config.json`.

```sh
k6 run --features native-histograms script.js
K6_FEATURES=native-histograms k6 run script.js
k6 features --json
```

Enabled flags are added to metric tags and retained in archives and Cloud
workers. `native-histograms`, the first flag, makes Trend metrics use
experimental native histograms. Use `k6 features` to inspect each flag and its
lifecycle rather than assuming an experimental flag remains available.
