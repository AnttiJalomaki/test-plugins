---
name: cloudflare-workers-knowledge-patch
description: Cloudflare Workers
version: null
license: MIT
metadata:
  author: Nevaberry
---



# Cloudflare Workers Knowledge Patch

## Reference index

| Reference | Topics |
|---|---|
| [Wrangler and deployment](references/wrangler-and-deployment.md) | Wrangler v4 migration, local and remote commands, authentication profiles, generated runtime types |
| [Vite development and testing](references/vite-development-and-testing.md) | Vite plugin configuration, auxiliary Workers, build deployment, production-build integration tests |
| [Runtime compatibility](references/runtime-compatibility.md) | Compatibility-date gates, Fetch and Cache behavior, JavaScript and stream APIs, tracing, email, Dynamic Workers |
| [Node.js compatibility](references/nodejs-compatibility.md) | `nodejs_compat`, process and environment behavior, modules and stubs, timers, performance, API corrections |
| [RPC and WebSockets](references/rpc-and-websockets.md) | RPC entrypoints and capabilities, ownership, pipelining, placement, message limits, WebSocket behavior |
| [Static Assets and Pages migration](references/static-assets-and-pages-migration.md) | Asset routing and bindings, Pages conversion, exclusions, builds, previews, headers, redirects, domains |

## Use this patch

1. Read the Wrangler reference before upgrading to Wrangler v4, scripting a
   resource command, selecting an account, or committing generated runtime
   types.
2. Read the runtime and Node.js references before changing a compatibility date
   or flag. A date change can alter request, module, stream, and serialization
   behavior together.
3. Read the RPC and WebSocket reference before passing stubs, streams, requests,
   responses, or application-defined objects across a binding.
4. Read the Static Assets reference before replacing Workers Sites or Pages.
   Asset-first routing and Worker-first middleware must be configured
   deliberately.
5. Read the Vite reference when local behavior must match `workerd`, when a
   build contains auxiliary Workers, or when tests need a production bundle.

## Breaking changes and migration priorities

### Migrate before adopting Wrangler v4

Wrangler v4 does not support Node.js 16 and follows the Node.js release
lifecycle. Its bundled esbuild moves from 0.17.19 to 0.24, wildcard dynamic
imports include every matching file, and later Wrangler minor releases may
change the pre-1.0 esbuild version.

Replace removed interfaces:

| Removed | Replacement |
|---|---|
| `legacy_assets` | Static Assets |
| `node_compat` | `nodejs_compat` |
| `getBindingsProxy()` | `getPlatformProxy()` |
| `publish` | `deploy` |
| `pages publish` | `pages deploy` |
| `generate` | `npm create cloudflare@latest` |
| `wrangler version` | `wrangler --version` |

Remove `usage_model`; it has no effect. Workers Sites and service environments
using `legacy_env` are deprecated in favor of Static Assets and Wrangler
environments.

### Make remote resource access explicit

Wrangler commands capable of both local and remote operation default to local.
Pass `--remote` when a KV or R2 script intends to access account data:

```sh
wrangler kv key get --binding MY_KV "my-key" --remote
```

### Audit behavior when advancing compatibility dates

These gates are especially likely to require code changes:

| Date | Behavior |
|---|---|
| `2025-04-01` | Bindings populate `process.env`; Static Assets navigation fallback precedes the Worker unless Worker-first routing applies |
| `2025-09-01` | Cross-origin redirects strip `Authorization`; end-of-life Node APIs are removed under `nodejs_compat` |
| `2025-12-03` | Optional runtime fields may exist with value `undefined` |
| `2026-01-20` | RPC parameter stubs are duplicated instead of transferred |
| `2026-01-22` | `require()` returns a default export when present |
| `2026-02-19` | Iterable request and response bodies stream instead of being coerced |
| `2026-03-03` | Rejection timing and WebSocket close-reason validation change |
| `2026-03-10` | WebSockets automatically answer Close frames |
| `2026-03-17` | WebSocket binary messages default to `Blob` |
| `2026-03-24` | Encoding streams and writable-writer backpressure behavior change |

Use the documented rollback flag only while adapting code; do not assume a
single flag restores unrelated gates from the same date.

### Replace Pages routing assumptions explicitly

Workers Static Assets does not infer SPA or 404 fallback from files. Configure
`not_found_handling`, and configure `run_worker_first` for authentication,
logging, APIs, or middleware that must run before assets:

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

Omit `binding` for an assets-only Worker. With a `main`, the Worker can delegate
to `env.ASSETS.fetch(request)`.

### Treat Node stubs as import compatibility only

Several `nodejs_compat` modules import successfully without exposing their host
facility. Do not infer functional child processes, worker threads, SQLite,
Inspector, or similar facilities from successful imports. Module-specific
enable and disable flags control individual stub rollouts.

### Update RPC ownership assumptions

RPC calls are asynchronous even when the callee is synchronous. Functions and
`RpcTarget` instances cross as capabilities, while streams, requests, and
responses transfer ownership.

```ts
using counter = await env.COUNTERS.create();
await counter.increment();
const value = await counter.value;
```

Since `2026-01-20`, a stub in call parameters is duplicated for the call, so
forwarding no longer disposes the caller's stub. A callee retaining a received
parameter stub beyond the call must still save `stub.dup()`.

### Adapt WebSocket close and binary handling

From `2026-03-10`, an incoming Close frame triggers an automatic reply and the
socket is already `CLOSED` before the `close` event. Use
`ws.accept({ allowHalfOpen: true })` only for an upgrade-created socket that
must preserve the old half-open phase.

From `2026-03-17`, set the type before accepting when `ArrayBuffer` is required:

```ts
ws.binaryType = "arraybuffer";
ws.accept();
```

Durable Object hibernatable WebSocket handlers continue receiving
`ArrayBuffer`.

## High-value configuration

### Generate types from actual Worker configuration

Run `wrangler types` to generate `worker-configuration.d.ts` from compatibility
dates, flags, bindings, and module rules. Include it with
`compilerOptions.types`, add `@types/node` for `nodejs_compat`, and detect drift
in CI:

```sh
wrangler types --check
```

The root of `@cloudflare/workers-types` v5 contains the latest stable types;
experimental APIs are under `/experimental`, and dated entrypoints are gone.

### Use the Workers Vite plugin for runtime-faithful development

```ts
import { cloudflare } from "@cloudflare/vite-plugin";
import { defineConfig } from "vite";

export default defineConfig({ plugins: [cloudflare()] });
```

Application code runs in `workerd` while Vite retains HMR. The plugin supports
SPA, SSR, static, and API applications. Deploy every auxiliary Worker
separately; deploying the entry Worker does not deploy the others.

### Select authentication intentionally

Named OAuth profiles can be activated for a directory tree:

```sh
wrangler auth create client-a
wrangler auth activate client-a ~/clients/client-a
wrangler deploy --profile client-a
```

Keep `account_id` when a project should be constrained to one account.
`CLOUDFLARE_API_TOKEN` takes precedence in automation.

## Implementation checklist

- Inspect `compatibility_date` and `compatibility_flags` before explaining
  runtime behavior.
- Separate Wrangler CLI locality from Vite remote-binding defaults.
- Distinguish implemented Node APIs from import-only stubs.
- Test optional fields by value, such as `obj.key !== undefined`, rather than
  with property-presence checks.
- Clone a `Request` or `Response`, or `tee()` a readable stream, before an RPC
  send when the caller still needs it.
- Deploy Vite auxiliary Workers from their generated `dist` configurations.
- Put `.assetsignore` inside the configured asset directory.
- Charge and latency-model Worker-first asset middleware as normal Worker
  invocations.
- Use `createTestHarness()` for tests against production-built Workers.

## Detailed guidance

- Read [Wrangler and deployment](references/wrangler-and-deployment.md) for the
  complete CLI migration, authentication, and type-generation behavior.
- Read [Vite development and testing](references/vite-development-and-testing.md)
  for plugin resolution, state, inspector, remote bindings, auxiliary builds,
  and the integration-test harness.
- Read [Runtime compatibility](references/runtime-compatibility.md) before
  changing dates or flags that affect fetch, cache, streams, JavaScript APIs,
  tracing, email, or Dynamic Workers.
- Read [Node.js compatibility](references/nodejs-compatibility.md) for the
  module matrix, process changes, environment population, stubs, timers, and
  Node-specific corrections.
- Read [RPC and WebSockets](references/rpc-and-websockets.md) for capabilities,
  ownership, forwarding, pipelining, placement, and socket protocol changes.
- Read [Static Assets and Pages migration](references/static-assets-and-pages-migration.md)
  for routing, project conversion, exclusions, builds, previews, and domains.
