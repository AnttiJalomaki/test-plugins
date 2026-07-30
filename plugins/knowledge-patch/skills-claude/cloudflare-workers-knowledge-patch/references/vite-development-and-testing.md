# Vite development and testing

## Workers Vite plugin

`@cloudflare/vite-plugin` v1 runs application code in `workerd` during Vite
development while preserving HMR. It supports SPA, SSR, static, and API
workloads (batch `2025`).

```ts
import { cloudflare } from "@cloudflare/vite-plugin";
import { defineConfig } from "vite";

export default defineConfig({
  plugins: [cloudflare()],
});
```

## Configuration resolution and defaults

The entry Worker configuration is resolved in this order:

1. the plugin's `configPath`;
2. `CLOUDFLARE_VITE_WRANGLER_CONFIG_PATH`;
3. a root `wrangler.jsonc`, `wrangler.json`, or `wrangler.toml`.

A plugin `config` object or callback is applied afterward.

Default development behavior:

| Setting | Default |
|---|---|
| persisted state | `.wrangler/state` |
| inspector port | `9229` |
| remote bindings | enabled |

Plugin options can override those defaults and can expose development through a
tunnel.

## Auxiliary Workers

Every `auxiliaryWorkers` entry must provide `configPath`, `config`, or both.
Requests still enter through the main Worker. Builds put each Worker in its own
`dist` subdirectory.

`wrangler deploy` deploys only the entry Worker. Deploy auxiliary Workers one
at a time from their generated configurations:

```sh
wrangler deploy -c dist/<auxiliary-worker>/wrangler.json
```

## Static Assets with the plugin

A project using the Cloudflare Vite plugin does not need to specify
`assets.directory`; the integration supplies the build relationship. Other
Static Assets routing options still matter where applicable.

## Production-build integration tests

Wrangler's `createTestHarness()` exercises Workers built by Wrangler or the
Cloudflare Vite plugin from any Node.js test runner (batch `2026`). It replaces
`unstable_startWorker()` and `unstable_dev()`.

The harness can:

- load multiple Worker configurations;
- listen and dispatch requests through `server.fetch()`;
- reset persisted storage with `server.reset()`;
- expose runtime logs;
- mock outbound requests;
- integrate with Playwright.

```ts
const server = createTestHarness({
  workers: [{ configPath: "./workers/api/wrangler.jsonc" }],
});

await server.listen();
const response = await server.fetch("http://api.example.com/");
await server.reset();
await server.close();
```

Always close the harness. Reset storage between cases that require isolation.
Because the harness runs production-built output, use it for behavior that a
source-only test or a non-`workerd` development server cannot validate.
