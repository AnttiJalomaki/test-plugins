# CLI, Configuration, and Execution

## Run TypeScript without a separate build

k6 directly executes `.ts` test files (since 1.0.0):

```typescript
import http from 'k6/http';

interface Target {
  url: string;
}

const target: Target = { url: 'https://quickpizza.grafana.com/' };

export default function () {
  http.get(target.url);
}
```

```sh
k6 run script.ts
```

## Configure end-of-test summaries

### Disable the human-readable summary

`--no-summary` and `K6_NO_SUMMARY` are deprecated. Use summary mode `disabled`
instead (since 1.3.0):

```sh
k6 run --summary-mode=disabled script.js
K6_SUMMARY_MODE=disabled k6 run script.js
```

The `legacy` summary mode is also deprecated. Migrate its consumers to
`compact` or `full`; legacy mode was planned for removal in v2 (1.3.0).

### Opt into the structured machine-readable shape

The newer structured shape is available to both `--summary-export` and
`handleSummary()` (since 1.5.0). It is planned to become the default in v2, so
update downstream parsing before opting in:

```sh
k6 run script.js --new-machine-readable-summary --summary-export=summary.json
```

Use `K6_NEW_MACHINE_READABLE_SUMMARY` as the environment equivalent.

## Use feature flags

Experimental behavior can be enabled for `k6 run` and `k6 cloud run` with
repeated or comma-separated `--features`, with `K6_FEATURES`, or with the
`features` key in `config.json` (since 2.1.0).

```sh
k6 run --features native-histograms script.js
K6_FEATURES=native-histograms k6 run script.js
k6 features --json
```

Enabled features are attached to metric tags and preserved in archives and
Cloud workers. Use `k6 features` or its JSON form to inspect flags and their
lifecycle. The first flag, `native-histograms`, makes trend metrics use
experimental native histograms.

## Understand `--vus` with configured scenarios

When a script defines scenarios, `k6 run script.js --vus N` now warns and
replaces them with a `shared-iterations` scenario containing `N` VUs and `N`
iterations (2.1.0). It is no longer silently ignored. Do not add `--vus` to a
scenario-based invocation unless replacement is intended.

## Enable the HTTP API intentionally

The k6 HTTP API does not listen on `localhost:6565` by default in v2 (2.0.0).
Enable it explicitly for control-plane clients:

```sh
k6 run --address=localhost:6565 script.js
```

`K6_ADDRESS` is the environment equivalent. Remove health checks or control
clients that assume the API always exists, or configure the address.

## Consume explicit-failure execution status

Code consuming execution status can distinguish a test explicitly marked as
failed by handling `ExecutionStatusMarkedAsFailed` (since 1.6.0). Preserve this
state in reporters instead of merging it with an infrastructure abort or a
successful completion.

## Diagnose HTTP call signatures

`http.get()` and `http.head()` warn when given extra positional arguments
(since 1.8.0). The extra values are still ignored. Treat the warning as a
signature bug and move supported options into the documented parameter object
instead of relying on ignored arguments.

## Diagnose automatic extension provisioning

Automatic extension provisioning logs artifact resolution, cache hits,
downloads, retries, and cache pruning through ordinary k6 log entries (1.8.0).
Preserve the relevant log levels when diagnosing a provisioned binary; a
separate provisioning trace is not required.
