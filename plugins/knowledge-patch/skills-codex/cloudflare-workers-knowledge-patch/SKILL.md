---
name: cloudflare-workers-knowledge-patch
description: Cloudflare Workers
version: null
license: MIT
metadata:
  author: Nevaberry
---


# Cloudflare Workers Knowledge Patch

Use this skill when writing, migrating, testing, or reviewing Cloudflare
Workers code and Wrangler configuration. Inspect the project's
`compatibility_date`, compatibility flags, Wrangler major version, asset
configuration, and bindings before applying date-gated behavior.

## Reference index

| Reference | Topics |
| --- | --- |
| [wrangler-tooling-and-testing.md](references/wrangler-tooling-and-testing.md) | Wrangler v4, Vite, authentication profiles, generated types, integration tests, observability |
| [runtime-and-web-platform.md](references/runtime-and-web-platform.md) | Request and stream behavior, cache controls, JavaScript APIs, execution contexts, serialization, email |
| [nodejs-compatibility.md](references/nodejs-compatibility.md) | `nodejs_compat`, `process.env`, modules and stubs, timers, performance, EOL APIs |
| [rpc-and-websockets.md](references/rpc-and-websockets.md) | RPC entrypoints and capabilities, ownership, pipelining, limits, WebSocket compatibility |
| [static-assets-and-pages-migration.md](references/static-assets-and-pages-migration.md) | Static Assets routing and binding, Pages conversion, builds, previews, domains |

## Breaking changes and migration traps

### Upgrade Wrangler v4 deliberately

- Run Wrangler v4 on a supported Node.js release; Node.js 16 is no longer
  supported.
- Expect bundling differences from the bundled esbuild move from 0.17.19 to
  0.24. Wildcard dynamic imports include every matching file.
- Pin and test Wrangler minor updates when bundling output matters because a
  minor may update pre-1.0 esbuild.
- Commands that can operate locally or remotely now choose local operation by
  default. Add `--remote` for account data, including KV and R2 operations.

Replace removed commands and configuration:

| Removed or deprecated | Use |
| --- | --- |
| `legacy_assets`, Workers Sites | Static Assets |
| `node_compat` | `nodejs_compat` |
| `getBindingsProxy()` | `getPlatformProxy()` |
| `publish` | `deploy` |
| `pages publish` | `pages deploy` |
| `generate` | `npm create cloudflare@latest` |
| `wrangler version` | `wrangler --version` |
| service environments with `legacy_env` | Wrangler environments |

Remove `usage_model`; it has no effect.

### Treat compatibility dates as runtime inputs

Do not infer behavior from package versions alone. Read the Worker's
compatibility date and flags, then check the relevant reference. Important
behavior changes include:

- `2025-04-01`: bindings can populate `process.env`; Static Assets navigation
  fallback can run before the Worker.
- `2025-06-16`: unknown import attributes throw, and cross-request use of
  AsyncLocalStorage snapshots and bound functions throws.
- `2025-09-01`: cross-origin redirects strip `Authorization`, and
  `nodejs_compat` rolls up end-of-life Node API removals.
- `2025-12-03`: optional runtime fields may exist with value `undefined`.
- `2026-01-22`: `require()` prefers a module's default export.
- `2026-02-19`: iterable request and response bodies stream instead of being
  stringified.
- `2026-03-17`: WebSocket binary messages default to `Blob`.

Use a rollback flag only while migrating; record why it exists and test both
sides before advancing the compatibility date.

### Migrate Pages routing explicitly

Static Assets does not infer SPA or custom-404 routing. Set
`assets.not_found_handling`, and use `assets.run_worker_first` for API,
authentication, logging, or middleware routes that must execute before an
asset response.

```jsonc
{
  "main": "src/index.js",
  "assets": {
    "directory": "./dist",
    "binding": "ASSETS",
    "not_found_handling": "single-page-application",
    "run_worker_first": ["/api/*", "!/api/docs/*"]
  }
}
```

Keep `_worker.js` outside the asset directory or exclude it. Compile a Pages
`functions/` directory into one Worker entrypoint before setting `main`.
Preserve the Pages compatibility date during conversion.

## High-value configuration

### Generate types from the actual Worker configuration

Run `wrangler types` to create `worker-configuration.d.ts` from compatibility
settings, bindings, and module rules. Include the file in
`compilerOptions.types`, add `@types/node` when using `nodejs_compat`, and run
`wrangler types --check` in CI.

The root of `@cloudflare/workers-types` v5 represents the latest stable API;
use `/experimental` for experimental APIs. Do not use removed dated package
entrypoints.

### Choose the narrowest Node.js mode

Use `nodejs_compat` for the broader Node surface. If only
`AsyncLocalStorage` is required, use `nodejs_als` instead.

```jsonc
{
  "compatibility_flags": ["nodejs_als"]
}
```

Import-only stubs make imports resolve but do not provide the underlying host
facility. A specific stub can be enabled early or disabled after rollout with
`enable_nodejs_<name>_module` or `disable_nodejs_<name>_module`.

### Build with the Workers Vite plugin

`@cloudflare/vite-plugin` runs application code in `workerd` during Vite
development while retaining HMR:

```ts
import { cloudflare } from "@cloudflare/vite-plugin";
import { defineConfig } from "vite";

export default defineConfig({ plugins: [cloudflare()] });
```

Requests enter through the main Worker. Each auxiliary Worker needs a config
path, an inline config, or both. Build output separates Workers into `dist`
subdirectories, and each auxiliary Worker must be deployed separately.

### Test production builds

Use `createTestHarness()` with output built by Wrangler or the Vite plugin.
It can load multiple configurations, dispatch through `server.fetch()`, reset
storage, expose logs, mock outbound requests, and integrate with Playwright.
It replaces `unstable_startWorker()` and `unstable_dev()`.

Always close the harness:

```ts
const server = createTestHarness({
  workers: [{ configPath: "./workers/api/wrangler.jsonc" }],
});
await server.listen();
const response = await server.fetch("http://api.example.com/");
await server.reset();
await server.close();
```

## Runtime quick reference

### Requests, cache, and redirects

- Enable `enable_request_signal` to expose incoming cancellation through
  `Request.signal`; enable `request_signal_passthrough` to propagate it through
  an outgoing `fetch()`.
- `fetch(url, { cache: "no-cache" })` can force conditional revalidation for
  external-origin subrequests.
- Use `cf.vary` to bypass unconfigured `Vary` headers or normalize selected
  request-header values for caching.
- Retain credentials on a cross-origin redirect only with the explicit
  `retain_authorization_on_cross_origin_redirect` escape hatch.

### Execution and object checks

- Every event-handler invocation receives a fresh `ctx`. Do not share it
  across requests.
- Check optional fields with `obj.key !== undefined`, not property-presence
  tests.
- Finalizers are nondeterministic, may run after a handler, have no async
  context, and cannot perform I/O or emit tail events.
- `eval()` and `new Function()` can be allowed during startup without allowing
  arbitrary dynamic evaluation during request handling.

### RPC

RPC requires compatibility date `2024-04-03` or later, or the `rpc` flag.
Public `WorkerEntrypoint` and Durable Object methods are asynchronous to the
caller even when their implementation is synchronous.

- Extend `RpcTarget` for application objects that cross RPC.
- Treat functions and `RpcTarget` values as remote capabilities.
- Use `using` or explicit disposal for stubs.
- Omit an intermediate `await` when promise pipelining should combine calls
  into one round trip.
- Clone requests or responses, or `tee()` readable streams, before sending
  them if the caller still needs them.
- Do not persist a stub forwarded through another Worker.
- Do not expect Smart Placement to affect RPC.

### WebSockets

- The maximum WebSocket message and JSRPC serialized message sizes are both
  32 MiB.
- Keep close reasons within 123 UTF-8 bytes.
- Use `accept({ allowHalfOpen: true })` only when a proxy requires the former
  half-open close phase.
- Set `binaryType = "arraybuffer"` before `accept()` when `ArrayBuffer` is
  required; the standard default is `"blob"`.
- Durable Object hibernatable handlers continue to receive `ArrayBuffer`.

## Review checklist

1. Confirm the Wrangler major and Node.js host version.
2. Read `compatibility_date` and all compatibility flags.
3. Distinguish local Wrangler operations from intentional `--remote` access.
4. Verify asset fallback and Worker-first route ordering.
5. Treat Node stub modules as import compatibility, not implemented services.
6. Audit ownership of RPC stubs, streams, requests, and responses.
7. Test WebSocket close and binary-message assumptions.
8. Regenerate runtime types and run the production-build harness.
9. Check deployment account/profile selection and auxiliary Worker deploys.
10. Prefer current project code, configuration, and test behavior when they
    establish a newer platform contract.
