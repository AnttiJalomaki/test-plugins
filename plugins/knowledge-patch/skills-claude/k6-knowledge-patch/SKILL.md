---
name: k6-knowledge-patch
description: Grafana k6
version: 2.1.0
license: MIT
metadata:
  author: Nevaberry
---


# Grafana k6 Knowledge Patch

Use this skill when writing, reviewing, migrating, or diagnosing Grafana k6
tests, extensions, browser scripts, Cloud workflows, and output configuration.
Check the project's pinned k6 version before applying version-sensitive advice.

## Reference index

| Reference | Topics |
| --- | --- |
| [Browser testing](references/browser-testing.md) | Locators, frames, routing, waits, events, redirects, proxies, and Cloud browser logs |
| [CLI, Cloud, and configuration](references/cli-cloud-and-configuration.md) | Commands, summaries, Cloud projects and tests, configuration, feature flags, containers, and the HTTP API |
| [Extensions and dependencies](references/extensions-and-dependencies.md) | Automatic resolution, dependency manifests, `k6 x`, Go requirements, DNS, and archive metadata |
| [Migrations and compatibility](references/migrations-and-compatibility.md) | Versioning guarantees, v2 removals, exit statuses, module paths, and HTTP argument warnings |
| [Outputs and observability](references/outputs-and-observability.md) | OpenTelemetry, Prometheus, Rate and Gauge shapes, summaries, dashboards, and native histograms |
| [Scripting, security, and protocols](references/scripting-security-and-protocols.md) | TypeScript, assertions, logging, WebSockets, gRPC, crypto, TOTP, secrets, and execution status |

## Migration triage

### Moving extensions to k6 v2

- Change Go imports from `go.k6.io/k6/...` to `go.k6.io/k6/v2/...`.
- Replace dependencies on generated `MarshalJSON` or `UnmarshalJSON` methods
  from public k6 Go types with standard `encoding/json`.
- Remove `K6_BINARY_PROVISIONING` and
  `K6_ENABLE_COMMUNITY_EXTENSIONS`. Automatic resolution now uses the default
  build service; set `K6_AUTO_EXTENSION_RESOLUTION` only to disable it.
- Expect provisioned `k6 x` commands to receive the host version in
  `K6_PROVISION_HOST_VERSION`.

### Replacing removed control and Cloud interfaces

- Replace the removed `externally-controlled` executor with an appropriate
  supported executor such as `ramping-vus`, `constant-vus`, or
  `constant-arrival-rate`. The `pause`, `resume`, `scale`, and `status`
  commands have no replacements.
- Replace `k6 login` with `k6 cloud login`, positional `k6 cloud script.js`
  with `k6 cloud run script.js`, and `--upload-only` with
  `k6 cloud upload script.js`.
- Move `options.ext.loadimpact` fields to `options.cloud`.
- Supply InfluxDB credentials through `K6_INFLUXDB_*` variables.
- Treat Cloud abort exit status `97` as failure. Threshold aborts remain `99`.

### Updating configuration and services

- Move legacy `{USER_CONFIG_DIR}/loadimpact/config.json` to
  `{USER_CONFIG_DIR}/k6/config.json`, or recreate it with `k6 cloud login`.
- Enable the HTTP API explicitly with `--address` or `K6_ADDRESS`; it no
  longer listens on `localhost:6565` by default.
- Remove `K6_OTEL_SINGLE_COUNTER_FOR_RATE`; the labeled single-counter Rate
  representation is mandatory.
- Use `--out=web-dashboard` for the built-in dashboard instead of requiring
  the separate dashboard extension.

## High-value scripting changes

### Run TypeScript directly

k6 accepts `.ts` entry files without a separate transpilation step:

```typescript
import http from 'k6/http';

interface Target { url: string }
const target: Target = { url: 'https://quickpizza.grafana.com/' };

export default function () {
  http.get(target.url);
}
```

```sh
k6 run script.ts
```

`k6/browser`, `k6/net/grpc`, and `k6/crypto` are stable modules. Import stable
WebSockets from `k6/websockets`; migrate away from
`k6/experimental/websockets`.

### Fail without stopping immediately

Use `exec.test.fail()` when the test must be marked failed while execution
continues. Consumers of execution status can distinguish this outcome through
`ExecutionStatusMarkedAsFailed`.

### Use supported HTTP signatures

`http.get()` and `http.head()` warn about extra positional arguments. The
values are still ignored, so repair the call rather than relying on them.

## Automatic extensions

k6 detects static ES module imports and provisions a matching binary. Dynamic
CommonJS `require()` calls are not discovered. For CommonJS, put a directive
at the start of every relevant file, after at most a shebang, whitespace, or
comments:

```javascript
"use k6 with k6/x/redis"
const redis = require('k6/x/redis');
```

Use `k6 deps --json script.js` to inspect detected dependencies and
`K6_DEPENDENCIES_MANIFEST` to constrain dependencies without a version pragma.
Use `k6 x` to discover extension subcommands; absent subcommand extensions can
be provisioned on demand.

Building current extension code requires Go 1.25 or newer. The default Go
toolchain is 1.26.

## Browser patterns

### Arm waits before their trigger

Start request and response waits concurrently with the action that triggers
them so fast traffic cannot be missed:

```javascript
const [response] = await Promise.all([
  page.waitForResponse(/\/api\/.*\.json$/),
  page.click('button[data-testid="load-data"]'),
]);
```

The same pattern applies to `page.waitForRequest()`. For named page events,
create the `page.waitForEvent()` promise before the triggering action.

### Prefer locator-native operations

- Use `count()`, `first()`, `nth(index)`, and `last()` for multi-match
  locators. `count()` returns immediately without a visibility wait.
- Use `filter({ hasText })` or `filter({ hasNotText })` for text filtering.
- Use `contentFrame()` or `frameLocator()` for iframe content while retaining
  locator retries.
- Use `pressSequentially()` when every keyboard event matters; use `fill()`
  for simple form filling.
- Use `page.route()` to abort, continue, or fulfill intercepted requests, and
  remove registrations with matching `unroute()` or with `unrouteAll()`.

## Cloud and CLI patterns

Use explicit Cloud subcommands:

```sh
k6 cloud login
k6 cloud run script.js
k6 cloud upload script.js
k6 cloud project list --format=json
k6 cloud test list --json
```

A login can save a default stack. Override it with `K6_CLOUD_STACK_ID` or
`options.cloud.stackID`. `k6 cloud test list` resolves its project from an
explicit project ID, then Cloud project configuration, then the configured
stack's default project.

For local Cloud execution, the Cloud secret source is enabled automatically;
use `--no-cloud-secrets` to opt out. Set `K6_CLOUD_PUSH_REF_ID` to reuse an
existing Cloud run.

`--vus N` now replaces configured scenarios with a `shared-iterations`
scenario of `N` VUs and `N` iterations, and emits a warning.

## Outputs and summaries

- Use `--summary-mode=disabled` instead of `--no-summary` or
  `K6_NO_SUMMARY`. Migrate `legacy` mode to `compact` or `full`.
- Opt in to the structured summary shape with
  `--new-machine-readable-summary` or `K6_NEW_MACHINE_READABLE_SUMMARY` when
  using `--summary-export` or `handleSummary()`.
- Use `--out opentelemetry`; the `experimental-opentelemetry` name and
  `K6_OTEL_EXPORTER_TYPE` are deprecated.
- Configure OpenTelemetry HTTP Basic Auth with
  `K6_OTEL_HTTP_EXPORTER_USERNAME` and `K6_OTEL_HTTP_EXPORTER_PASSWORD`.
- Treat exported Rate values as a counter labeled `zero` or `nonzero`.
- Replace browser FID thresholds with INP thresholds.

## Feature flags

Pass repeated or comma-separated `--features` values, set `K6_FEATURES`, or
use the `features` configuration key. Inspect flags with `k6 features` or
`k6 features --json`. Feature state is attached to metric tags and retained
in archives and Cloud workers. `native-histograms` makes Trend metrics use
experimental native histograms.

## Verify version-sensitive behavior

Before changing a test or extension:

1. Identify the exact k6 version used locally, in CI, in Cloud, and in the
   container image.
2. Read the topic reference for the affected API or workflow.
3. Distinguish stable APIs from experimental or preview behavior.
4. Check environment variables, archived dependency metadata, and Cloud
   defaults that can change behavior outside the script.
5. Run the narrowest representative test and inspect logs, metrics, summary
   output, and process exit status.
