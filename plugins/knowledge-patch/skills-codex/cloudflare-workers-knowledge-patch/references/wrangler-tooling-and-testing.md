# Wrangler, Vite, Types, and Testing

Use this reference for CLI migrations, development configuration, deployment,
authentication, generated types, integration tests, and observability.
It incorporates the versioned source batches `2025` and `2026`.

## Wrangler v4 migration

Wrangler v4 follows the Node.js release lifecycle and drops Node.js 16. Its
bundled esbuild moves from 0.17.19 to 0.24. Wildcard dynamic imports bundle
every matching file, and Wrangler minor releases may update pre-1.0 esbuild,
so test bundle composition when upgrading even a minor Wrangler release.

Commands supporting both local and remote resources now default to local
operation. Add `--remote` when a KV or R2 command must touch account data:

```sh
wrangler kv key get --binding MY_KV "my-key" --remote
```

Wrangler v4 removes previously deprecated interfaces:

| Old interface | Replacement |
| --- | --- |
| `legacy_assets` | Static Assets |
| `node_compat` | `nodejs_compat` |
| `getBindingsProxy()` | `getPlatformProxy()` |
| `publish` | `deploy` |
| `pages publish` | `pages deploy` |
| `generate` | `npm create cloudflare@latest` |
| `wrangler version` | `wrangler --version` |

Remove `usage_model`, which is ineffective. Workers Sites is deprecated in
favor of Static Assets. Service environments using `legacy_env` are
deprecated in favor of Wrangler environments.

## Workers Vite plugin

`@cloudflare/vite-plugin` v1 runs application code in `workerd` during Vite
development and retains HMR. It supports SPA, SSR, static, and API workloads.

```ts
import { cloudflare } from "@cloudflare/vite-plugin";
import { defineConfig } from "vite";

export default defineConfig({ plugins: [cloudflare()] });
```

The entry Worker config is resolved in this order:

1. The plugin's `configPath`.
2. `CLOUDFLARE_VITE_WRANGLER_CONFIG_PATH`.
3. A root `wrangler.jsonc`, `wrangler.json`, or `wrangler.toml`.
4. The plugin's `config` object or callback is applied afterward.

By default, state persists in `.wrangler/state`, the inspector listens on
port 9229, and remote bindings are enabled. Plugin options can override these
defaults or expose development through a tunnel.

Each `auxiliaryWorkers` entry requires `configPath`, `config`, or both.
Requests still enter through the main Worker. Builds put Workers in distinct
`dist` subdirectories. `wrangler deploy` deploys only the entry Worker, so
deploy each auxiliary Worker independently:

```sh
wrangler deploy -c dist/<auxiliary-worker>/wrangler.json
```

## Authentication profiles

Wrangler supports named OAuth logins whose activation applies to a directory
and its descendants:

```sh
wrangler auth create client-a
wrangler auth activate client-a ~/clients/client-a
wrangler deploy --profile client-a
```

`account_id` can still constrain a project to the intended account.
`CLOUDFLARE_API_TOKEN` takes precedence over profiles in CI or other automated
environments.

## Configuration-derived types

`wrangler types` generates `worker-configuration.d.ts` from the Worker's
compatibility date, flags, bindings, and module rules. Add that file to
`compilerOptions.types`. Add `@types/node` when `nodejs_compat` is active.
Run `wrangler types --check` in CI when the output is committed.

`@cloudflare/workers-types` v5 exposes only current stable types at its root
and experimental APIs from `/experimental`; its dated entrypoints are removed.

## Production-build integration tests

`createTestHarness()` runs Workers produced by Wrangler or the Vite plugin
from any Node.js test runner. It replaces `unstable_startWorker()` and
`unstable_dev()`. A harness can load multiple Worker configs, dispatch through
`server.fetch()`, reset storage with `server.reset()`, surface runtime logs,
mock outbound requests, and support Playwright.

```ts
const server = createTestHarness({
  workers: [{ configPath: "./workers/api/wrangler.jsonc" }],
});
await server.listen();
const response = await server.fetch("http://api.example.com/");
await server.reset();
await server.close();
```

Use `try`/`finally` in real tests so the server is closed after failures.

## Tracing and tail visibility

From compatibility date `2025-11-05`, enabling observability also enables
automatic Workers tracing:

```jsonc
{
  "observability": { "enabled": true }
}
```

For older compatibility dates, opt in explicitly:

```jsonc
{
  "observability": { "traces": { "enabled": true } }
}
```

Preview warnings once restricted to DevTools are also delivered to an
attached tail Worker.
