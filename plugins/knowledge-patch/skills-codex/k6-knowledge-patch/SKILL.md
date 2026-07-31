---
name: k6-knowledge-patch
description: Grafana k6
version: 2.1.0
license: MIT
metadata:
  author: Nevaberry
---


# Grafana k6 Development Guidance

Use this skill when writing, reviewing, upgrading, or operating k6 tests. Start
with migration guidance for an existing project, then open the topic reference
that matches the work. Prefer explicit commands and current module paths over
compatibility switches that only postpone a migration.

## Reference index

| Reference | Topics |
| --- | --- |
| [Browser testing](references/browser-testing.md) | Locators, frames, request routing, page events, proxies, browser metrics, and Cloud log filtering |
| [CLI, configuration, and execution](references/cli-config-and-execution.md) | TypeScript, summaries, feature flags, execution status, HTTP API, VU overrides, and diagnostics |
| [Cloud and secrets](references/cloud-and-secrets.md) | Cloud commands, stack and project resolution, exit status, local execution, and secret sources |
| [Extensions and dependencies](references/extensions-and-dependencies.md) | Automatic provisioning, dependency discovery, `k6 x`, official extensions, archives, and extension runtime integration |
| [Metrics, outputs, and runtime](references/metrics-outputs-and-runtime.md) | OpenTelemetry, Prometheus, Rate and Gauge shapes, WebSockets, gRPC, crypto, assertions, and logging |
| [Migration and compatibility](references/migration-and-compatibility.md) | Major-version removals, Go APIs and toolchains, Docker behavior, support policy, and upgrade audit |

## Start with breaking changes

### Audit a v2 migration

For k6 v2, address all of these before debugging secondary failures:

- Change Go imports from `go.k6.io/k6/...` to `go.k6.io/k6/v2/...`.
- Replace the removed `externally-controlled` executor. The `pause`, `resume`,
  `scale`, and `status` commands no longer exist.
- Use `k6 cloud login`, `k6 cloud run`, and `k6 cloud upload`; remove the old
  positional Cloud form and `--upload-only`.
- Move `options.ext.loadimpact` to `options.cloud`.
- Remove `K6_OTEL_SINGLE_COUNTER_FOR_RATE`, `K6_BINARY_PROVISIONING`, and
  `K6_ENABLE_COMMUNITY_EXTENSIONS`.
- Move legacy configuration to `{USER_CONFIG_DIR}/k6/config.json`; v2 does not
  read the old `loadimpact` path.
- Enable the HTTP API explicitly with `--address` or `K6_ADDRESS` if automation
  expects it.
- Treat Cloud exit status `97` as failure. Threshold aborts remain `99`.

Read [Migration and compatibility](references/migration-and-compatibility.md)
for extension JSON changes, Docker tags, and build requirements. Read
[Cloud and secrets](references/cloud-and-secrets.md) before changing Cloud CI.

### Replace deprecated interfaces

- Use summary mode `disabled` instead of `--no-summary` or `K6_NO_SUMMARY`.
  Migrate `legacy` summary users to `compact` or `full`.
- Use `--out opentelemetry` and `K6_OTEL_EXPORTER_PROTOCOL`; the experimental
  output name and `K6_OTEL_EXPORTER_TYPE` are deprecated.
- Import WebSockets from `k6/websockets`, not
  `k6/experimental/websockets`.
- Replace `k6/experimental/redis` with the official Redis extension.
- Replace `browser_web_vital_fid` thresholds and integrations with
  `browser_web_vital_inp`.

## High-value workflows

### Run TypeScript directly

k6 runs `.ts` tests without a separate transpilation step:

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

### Inspect and provision dependencies

Use static ES module imports so automatic extension resolution and dependency
inspection can see dependencies:

```sh
k6 deps --json script.js
K6_DEPENDENCIES_MANIFEST='{"k6/x/faker":">=v0.4.4"}' k6 run script.js
```

Dynamic `require()` calls are not discovered. For unavoidable CommonJS, place a
`"use k6 with ..."` directive at the start of every relevant file. See
[Extensions and dependencies](references/extensions-and-dependencies.md) for
community-extension rules, archive metadata, and `k6 x` provisioning.

### Arm browser waits before actions

Start a page wait together with the action that triggers it:

```javascript
await Promise.all([
  page.waitForResponse(/\/api\/.*\.json$/),
  page.click('button[data-testid="load-data"]'),
]);
```

Use `waitForRequest()` for outgoing requests, `waitForEvent()` for named page
events, and `requestfailed` or `requestfinished` listeners for lifecycle
observation. Use `route()` to abort, modify, or fulfill traffic. The complete
locator, frame, routing, proxy, and event guidance is in
[Browser testing](references/browser-testing.md).

### Use structured summaries deliberately

The newer machine-readable summary is opt-in:

```sh
k6 run script.js --new-machine-readable-summary --summary-export=summary.json
```

It applies to both `--summary-export` and `handleSummary()` and is intended to
become the default in a later major line. Update consumers before enabling it.

### Resolve Cloud context explicitly

Save a default stack at login or override it for a run:

```sh
k6 cloud login --token "$MY_TOKEN" --stack my-stack-slug
K6_CLOUD_STACK_ID=12345 k6 cloud run script.js
```

For deterministic automation, pass project IDs explicitly. `k6 cloud test
list` otherwise resolves a project from environment or configuration and then
the configured stack. See [Cloud and secrets](references/cloud-and-secrets.md)
for exact precedence and local-execution behavior.

### Enable experimental behavior visibly

Feature flags can come from repeated or comma-separated `--features`,
`K6_FEATURES`, or `config.json`:

```sh
k6 run --features native-histograms script.js
k6 features --json
```

Enabled features become metric tags and survive archives and Cloud workers.
Treat `native-histograms` as experimental and make downstream metric consumers
understand its trend representation.

## Output and metric checks

- Exported Rate metrics use one counter labeled with `zero` and `nonzero`.
- OpenTelemetry uses the stable `opentelemetry` output name. Its HTTP exporter
  accepts Basic Auth through the documented username and password settings.
- Experimental Prometheus remote write defaults to TLS 1.3 and exposes a
  minimum-version override.
- Cloud output v2 reports Gauge `min` and `max` in their proper fields.
- The built-in dashboard runs with `--out=web-dashboard`.

Open [Metrics, outputs, and runtime](references/metrics-outputs-and-runtime.md)
when changing collectors, dashboards, WebSockets, gRPC payloads, cryptography,
or log parsing.

## Browser selection checklist

- Use `count()` when visibility waiting would be wrong.
- Use `first()`, `nth(index)`, and `last()` to select among locator matches.
- Use `filter({ hasText })` or `filter({ hasNotText })` to narrow a locator.
- Use `contentFrame()` or chain `frameLocator()` calls for iframe content.
- Use `pressSequentially()` when every keyboard event matters; `fill()` is for
  simple form filling.
- Configure a proxy per isolated browser context with
  `browser.newContext({ proxy: ... })`.

## Extension checklist

1. Prefer static ES module imports.
2. Run `k6 deps --json` and inspect inferred versions.
3. Add `K6_DEPENDENCIES_MANIFEST` constraints when no version pragma exists.
4. Let automatic resolution provision missing supported extensions.
5. Run `k6 x` to discover registry and compiled-in subcommands.
6. For Go extensions on v2, update the module path and use `encoding/json`
   instead of removed easyjson-generated methods.

## Upgrade verification

After migration, verify behavior at system boundaries:

- Run browser tests containing redirects and confirm request samples are not
  duplicated.
- Check Rate and Gauge queries rather than assuming their old export shapes.
- Exercise Cloud abort paths and assert exit status handling.
- Confirm the HTTP API is intentionally enabled or intentionally absent.
- Run extension discovery from the same archive or source used in deployment.
- Pin a Docker major-line tag when `latest` is too broad for the deployment.

Use the references as the source of detail; this file is the triage and
high-frequency command guide.
