---
name: k6-knowledge-patch
description: Grafana k6
version: 2.1.0
license: MIT
metadata:
  author: Nevaberry
---


# Grafana k6 Knowledge Patch

Use this skill when writing, reviewing, migrating, or operating Grafana k6
tests. Check the actual `k6 version`, container tag, and Cloud execution mode
before applying version-dependent advice. Project code, manifests, and observed
runtime behavior take precedence when they disagree with this guidance.

## Reference index

| Reference | Topics |
| --- | --- |
| [migrations-and-compatibility.md](references/migrations-and-compatibility.md) | v2 migration, removals, deprecations, Go extensions, containers, support policy |
| [cli-cloud-and-configuration.md](references/cli-cloud-and-configuration.md) | CLI parsing, config files, summaries, Cloud commands, feature flags, API and dashboard |
| [extensions-and-dependencies.md](references/extensions-and-dependencies.md) | automatic resolution, dependency discovery, `k6 x`, archives, extension DNS |
| [browser-testing.md](references/browser-testing.md) | locators, frames, events, request routing, redirects, proxies, Cloud browser logs |
| [scripting-security-and-protocols.md](references/scripting-security-and-protocols.md) | TypeScript, crypto, secrets, execution status, gRPC, WebSockets, logging, HTTP |
| [outputs-and-observability.md](references/outputs-and-observability.md) | machine-readable summaries, OpenTelemetry, Prometheus, Rate and Gauge changes |

## Start with breaking changes

### Migrating to k6 v2

Treat a major-version upgrade as a code and automation migration:

1. Change Go imports from `go.k6.io/k6/...` to
   `go.k6.io/k6/v2/...`.
2. Replace the removed `externally-controlled` executor and the removed
   `pause`, `resume`, `scale`, and `status` commands.
3. Use `k6 cloud login`, `k6 cloud run`, and `k6 cloud upload`; move
   `options.ext.loadimpact` to `options.cloud`.
4. Remove `K6_BINARY_PROVISIONING`, `K6_ENABLE_COMMUNITY_EXTENSIONS`, and
   `K6_OTEL_SINGLE_COUNTER_FOR_RATE`.
5. Move legacy config files to `{USER_CONFIG_DIR}/k6/config.json`; the old
   `loadimpact` path is no longer read.
6. Enable the HTTP API explicitly with `--address` or `K6_ADDRESS`.
7. Make CI treat Cloud exit status `97` as failure.
8. Replace dependencies on easyjson-generated methods with `encoding/json`.

Read [migrations-and-compatibility.md](references/migrations-and-compatibility.md)
before changing an extension, container deployment, or Cloud pipeline.

### Deprecations to remove from new code

| Avoid | Use |
| --- | --- |
| `k6/experimental/webcrypto` | global `crypto` |
| `--no-summary`, `K6_NO_SUMMARY` | `--summary-mode=disabled`, `K6_SUMMARY_MODE=disabled` |
| `--summary-mode=legacy` | `compact` or `full` |
| `experimental-opentelemetry` | `opentelemetry` |
| `K6_OTEL_EXPORTER_TYPE` | `K6_OTEL_EXPORTER_PROTOCOL` |
| `k6/experimental/redis` | the official Redis extension |
| `k6/experimental/websockets` | `k6/websockets` |
| `browser_web_vital_fid` | `browser_web_vital_inp` |

## High-value workflows

### Run JavaScript or TypeScript directly

k6 accepts `.ts` files directly. Stable modules include `k6/browser`,
`k6/net/grpc`, and `k6/crypto`. The JavaScript runtime also supports logical
assignment and destructuring in exports.

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

### Let k6 resolve extensions

Use static ES module imports so k6 can detect dependencies:

```javascript
import dns from 'k6/x/dns';
```

Inspect them before execution when needed:

```sh
k6 deps --json script.js
k6 x
```

Dynamic `require()` calls are not discovered. For CommonJS, put a
`"use k6 with ..."` directive at the beginning of every relevant file.
See [extensions-and-dependencies.md](references/extensions-and-dependencies.md)
for manifests, community extensions, archives, and provisioned subcommands.

### Synchronize browser actions with network activity

Arm waits before the action that triggers them:

```javascript
const [response] = await Promise.all([
  page.waitForResponse(/\/api\/.*\.json$/),
  page.click('button[data-testid="load-data"]'),
]);
```

The same pattern applies to `waitForRequest()` and event promises. Use
`page.route()` to abort, continue, or fulfill matching requests, and remove
handlers with the identical matcher via `unroute()` or all at once via
`unrouteAll()`.

### Work with locators and frames

Locators support positional selection, text filtering, direct counts,
page-context evaluation, sequential key events, and label-based option
selection. Use `contentFrame()` or chain `frameLocator()` calls for iframes:

```javascript
const frame = page.frameLocator('#payment-iframe');
await frame.locator('#card-number').fill('4242424242424242');
```

Locator actionability retries temporarily hidden targets. See
[browser-testing.md](references/browser-testing.md) for precise event,
redirect, routing, and proxy behavior.

### Retrieve secrets without exposing them

`k6/secrets` is asynchronous, and configured secret values are redacted from
logs:

```javascript
import secrets from 'k6/secrets';

export default async function () {
  const token = await secrets.get('token');
}
```

Configure a source with repeated `--secret-source` flags or
`K6_SECRET_SOURCE`. Cloud local execution enables the Cloud source
automatically unless `--no-cloud-secrets` is set. URL sources have only a mock
implementation, so do not treat them as a production secret-manager
integration.

### Mark a test failed and continue cleanup

`execution.test.fail()` records explicit failure without stopping execution:

```javascript
import exec from 'k6/execution';

export default function () {
  exec.test.fail('validation failed');
  // Metrics and cleanup can continue.
}
```

Execution-status consumers can identify this state as
`ExecutionStatusMarkedAsFailed`.

### Choose end-of-test output deliberately

The human summary modes are `compact`, `full`, and `disabled`. Threshold
values appear even when omitted from `summaryTrendStats`.

The newer structured shape for `--summary-export` and `handleSummary()` is
opt-in:

```sh
k6 run script.js --new-machine-readable-summary \
  --summary-export=summary.json
```

Read [outputs-and-observability.md](references/outputs-and-observability.md)
before changing parsers or metric backends.

### Configure Cloud identity explicitly

Save a default stack at login and override it per run with
`K6_CLOUD_STACK_ID` or `options.cloud.stackID`:

```sh
k6 cloud login --token "$MY_TOKEN" --stack my-stack-slug
k6 cloud run script.js
```

Use `k6 cloud project list` and `k6 cloud test list` for discovery. Project
resolution precedence and local-run reuse are detailed in
[cli-cloud-and-configuration.md](references/cli-cloud-and-configuration.md).

### Enable experimental behavior explicitly

Pass repeated or comma-separated `--features`, set `K6_FEATURES`, or use the
`features` config key. Inspect availability and lifecycle with:

```sh
k6 features --json
```

`native-histograms` makes Trend metrics use experimental native histograms.
Feature selections propagate into metric tags, archives, and Cloud workers.

