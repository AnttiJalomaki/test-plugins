---
name: k6-knowledge-patch
description: Grafana k6
version: 2.1.0
license: MIT
metadata:
  author: Nevaberry
---


# Grafana k6

Use this skill when writing, reviewing, migrating, or operating Grafana k6
tests. Determine the installed or pinned k6 version first, then apply only
guidance introduced at or below that version. Treat the project's scripts,
configuration, manifests, tests, and observed behavior as authoritative.

## Reference index

| Reference | Topics |
| --- | --- |
| [migration-and-compatibility.md](references/migration-and-compatibility.md) | v2 migration, removals, deprecations, Go builds, containers, configuration paths |
| [cli-config-and-execution.md](references/cli-config-and-execution.md) | CLI syntax, summaries, templates, execution status, feature flags, HTTP API |
| [browser-testing.md](references/browser-testing.md) | locators, frames, routing, waits, network events, proxies, browser metrics |
| [extensions-and-dependencies.md](references/extensions-and-dependencies.md) | automatic resolution, dependency manifests, `k6 x`, archives, extension APIs |
| [cloud-and-secrets.md](references/cloud-and-secrets.md) | Cloud commands, stacks, projects and tests, local execution, secret sources |
| [metrics-outputs-and-runtime.md](references/metrics-outputs-and-runtime.md) | OpenTelemetry, Prometheus, metric shapes, TypeScript, crypto, gRPC, WebSockets, logging |

## Start with breaking changes

### Migrating to k6 v2

Apply these changes together when moving a project or extension to v2:

- Change Go imports from `go.k6.io/k6/...` to `go.k6.io/k6/v2/...`.
- Replace the removed `externally-controlled` executor. The `pause`, `resume`,
  `scale`, and `status` commands have no replacements.
- Use `k6 cloud login`, `k6 cloud run`, and `k6 cloud upload`; the top-level
  `k6 login`, positional `k6 cloud script.js`, and `--upload-only` are gone.
- Move script configuration from `options.ext.loadimpact` to `options.cloud`.
- Remove `K6_OTEL_SINGLE_COUNTER_FOR_RATE`, `K6_BINARY_PROVISIONING`, and
  `K6_ENABLE_COMMUNITY_EXTENSIONS`.
- Move the legacy `loadimpact/config.json` file to `k6/config.json` or log in
  again; v2 no longer reads or migrates the old path.
- Opt in to the HTTP API with `--address` or `K6_ADDRESS`; it no longer listens
  on port 6565 by default.
- Treat Cloud exit status `97` as failure. A successful run is `0`, while a
  threshold abort remains `99`.
- Replace easyjson-specific calls on public Go types with `encoding/json`.

See
[migration-and-compatibility.md](references/migration-and-compatibility.md)
for container tags, built-in dashboard behavior, and the full compatibility
checklist.

### Resolve active deprecations

- Use `--summary-mode=disabled` instead of `--no-summary` or
  `K6_NO_SUMMARY`.
- Replace `legacy` summaries with `compact` or `full`.
- Replace `browser_web_vital_fid` thresholds and integrations with
  `browser_web_vital_inp`.
- Use `--out opentelemetry` and
  `K6_OTEL_EXPORTER_PROTOCOL`, not the experimental output name or
  `K6_OTEL_EXPORTER_TYPE`.
- Use `k6/websockets`, not `k6/experimental/websockets`.
- Migrate `k6/experimental/redis` to the official Redis extension.
- Use the global `crypto`, not `k6/experimental/webcrypto`.

## Extension resolution

k6 detects statically imported extensions and can provision a matching binary.
Prefer ES module imports:

```javascript
import dns from 'k6/x/dns';
```

Discovery does not follow dynamic or CommonJS `require()` calls. For required
CommonJS code, put a directive at the beginning of each relevant file:

```javascript
"use k6 with k6/x/redis"
const redis = require('k6/x/redis');
```

Use `k6 deps --json script.js` to inspect detected dependencies and
`K6_DEPENDENCIES_MANIFEST` to constrain dependencies with no version pragma.
Community extensions resolve through the default service in v2; set
`K6_AUTO_EXTENSION_RESOLUTION` only when resolution must be disabled.

Extension commands live below `k6 x`. Running `k6 x` discovers available
commands, and missing subcommand extensions can be provisioned automatically.
See [extensions-and-dependencies.md](references/extensions-and-dependencies.md)
before debugging provisioning, archives, or extension Go code.

## Browser synchronization patterns

Arm a network wait before the action that triggers it:

```javascript
const [response] = await Promise.all([
  page.waitForResponse(/\/api\/items/),
  page.click('#load'),
]);
```

The same pattern applies to `page.waitForRequest()`. For a general page event,
create the `page.waitForEvent()` promise before the action and await it
afterward.

Use locator APIs for retryable browser work:

- `count()` reports the current match count without waiting for visibility.
- `first()`, `nth(index)`, and `last()` select a positional match.
- `filter({ hasText, hasNotText })` narrows by text.
- `contentFrame()` and `frameLocator()` retain locator behavior in iframes.
- `pressSequentially()` emits the full keyboard-event sequence per character.
- `evaluate()` returns a value; `evaluateHandle()` returns a `JSHandle`.

Use `page.route()` to abort, continue, or fulfill matching requests. Remove the
same matcher with `page.unroute(matcher)`, or clear all routes with
`page.unrouteAll()`.

Read [browser-testing.md](references/browser-testing.md) for lifecycle events,
redirect behavior, per-context proxies, option labels, DOM ordering, and
browser observability.

## Cloud command shape

Use explicit Cloud subcommands:

```sh
k6 cloud login --token "$MY_TOKEN" --stack my-stack
k6 cloud run script.js
k6 cloud upload script.js
```

A saved default stack supplies a default project. Override the stack with
`K6_CLOUD_STACK_ID` or `options.cloud.stackID`. List resources with:

```sh
k6 cloud project list --format=json
k6 cloud test list --json
```

Local Cloud execution automatically exposes the built-in Cloud secret source;
pass `--no-cloud-secrets` to opt out. It can reuse an existing run when
`K6_CLOUD_PUSH_REF_ID` is set.

See [cloud-and-secrets.md](references/cloud-and-secrets.md) for project
precedence, custom binaries, secret-source syntax, redaction, and Cloud
failure semantics.

## Summaries and outputs

The human summary modes are:

- `compact`, the default concise form.
- `full`, with detailed metrics plus group and scenario results.
- `disabled`, for no end-of-test summary.

The old `legacy` mode is deprecated. A separate opt-in machine-readable shape
applies to `--summary-export` and `handleSummary()`:

```sh
k6 run script.js --new-machine-readable-summary --summary-export=summary.json
```

For OpenTelemetry, use the stable output:

```sh
k6 run --out opentelemetry script.js
```

Rate metrics use a single counter labeled `zero` or `nonzero`. Update
downstream queries accordingly. See
[metrics-outputs-and-runtime.md](references/metrics-outputs-and-runtime.md) for
TLS floors, Basic Auth, Cloud Gauge extrema, redirect samples, and runtime
features.

## Script and execution essentials

k6 runs `.ts` files directly:

```sh
k6 run script.ts
```

The JavaScript runtime supports logical assignment and destructuring in
exports. Web Crypto is available through global `crypto`. The stable protocol
modules include `k6/browser`, `k6/net/grpc`, `k6/crypto`, and
`k6/websockets`.

Use `execution.test.fail(message)` when a run must be marked failed but should
continue collecting metrics and performing cleanup. Consumers can recognize
that state as `ExecutionStatusMarkedAsFailed`.

Feature flags can be supplied through repeated or comma-separated
`--features`, `K6_FEATURES`, or `config.json`. Inspect them with
`k6 features --json`; `native-histograms` makes trend metrics use experimental
native histograms.

Be deliberate with CLI overrides:

- Repeat `--tag` for multiple tags; commas belong to a tag value.
- Extra positional arguments to `http.get()` and `http.head()` are ignored
  with a warning.
- `--vus N` replaces configured scenarios with a `shared-iterations` scenario
  containing `N` VUs and `N` iterations.

## Version-aware review

When reviewing an existing project:

1. Find the exact k6 binary or container tag used by local development and CI.
2. Check every option, environment variable, import, output name, and Cloud
   command against the applicable reference.
3. For browser tests, verify waits are armed before triggering actions and
   redirect-sensitive handlers account for the complete chain.
4. For extensions, inspect static imports with `k6 deps` and confirm archive
   metadata preserves dependency constraints.
5. Run the project's normal tests and a representative k6 invocation; prefer
   actual command output over assumptions.
