# CLI, configuration, and execution

## Run arguments and overrides

Since 1.0.0-rc1, a comma is part of a `--tag` value rather than a separator
between tag sets. Repeat the flag:

```sh
k6 run --tag 'label=a,b' --tag env=test script.js
```

Since 1.8.0, `http.get()` and `http.head()` warn about extra positional
arguments. The values remain ignored, but the warning indicates an unsupported
method signature.

Since 2.1.0, `k6 run script.js --vus N` warns and replaces script-defined
scenarios with one `shared-iterations` scenario containing `N` VUs and `N`
iterations. It is no longer silently ignored.

## Human-readable summaries

The redesigned summary introduced in 1.0.0-rc1 has these modes:

- `compact` is the default.
- `full` includes detailed metrics plus per-group and per-scenario results.
- `legacy` restores the older format, but was deprecated in 1.3.0 for removal
  in v2.
- `disabled`, added in 1.3.0, replaces deprecated `--no-summary` and
  `K6_NO_SUMMARY`.

```sh
k6 run --summary-mode=full script.js
K6_SUMMARY_MODE=disabled k6 run script.js
```

The 1.0.0-rc1 redesign did not change the `handleSummary` input or
`--summary-export` data structure. Since 1.0.0-rc2, summaries show threshold
values even when the values are absent from `summaryTrendStats`.

## Machine-readable summaries

Since 1.5.0, a new structured summary shape is available to
`--summary-export` and `handleSummary()`. Opt in with
`--new-machine-readable-summary` or `K6_NEW_MACHINE_READABLE_SUMMARY`; it was
planned to become the default in v2.

```sh
k6 run script.js --new-machine-readable-summary --summary-export=summary.json
```

Do not confuse this opt-in data shape with the human `compact`, `full`, or
`disabled` modes.

## Templates and script startup

Since 1.0.0-rc1, `k6 new` accepts a Go-template file. The template receives
`ScriptName` and `ProjectID`:

```sh
k6 new --template /path/to/my-template.js
```

Since 1.0.0, `.ts` scripts run directly without a separate transpilation step:

```sh
k6 run script.ts
```

## Marking a test failed

Since 1.0.0-rc2, `execution.test.fail()` marks the test failed but allows
execution, metric collection, and cleanup to continue:

```javascript
import exec from 'k6/execution';

export default function () {
  exec.test.fail('validation failed');
}
```

Since 1.6.0, status consumers can distinguish this outcome through
`ExecutionStatusMarkedAsFailed`.

## HTTP API server

Since 2.0.0, the HTTP API is opt-in rather than listening on
`localhost:6565` automatically:

```sh
k6 run --address=localhost:6565 script.js
```

`K6_ADDRESS` is the environment-variable equivalent.

## Feature flags

Since 2.1.0, experimental behavior can be enabled for `k6 run` and
`k6 cloud run` with:

- Repeated or comma-separated `--features` flags.
- `K6_FEATURES`.
- A `features` key in `config.json`.

Enabled features become metric tags and are preserved in archives and Cloud
workers. Use `k6 features` or `k6 features --json` to inspect flags and their
lifecycle. The first flag, `native-histograms`, makes trend metrics use
experimental native histograms:

```sh
k6 run --features native-histograms script.js
K6_FEATURES=native-histograms k6 run script.js
```
