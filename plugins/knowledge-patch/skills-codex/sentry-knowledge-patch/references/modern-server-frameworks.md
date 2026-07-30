# Modern server framework integrations

Use this reference to configure Sentry for Elysia, Hono, Nitro, and TanStack
Start, including error capture, source maps, deployment preloads, middleware
ordering, request data controls, and event tunneling.

## Shared data-collection control (modern-server-frameworks)

All four SDKs accept `dataCollection` controls for automatically enriched
request data. Disable default user information and all request-body capture
explicitly when needed:

```js
Sentry.init({
  dataCollection: { userInfo: false, httpBodies: [] },
});
```

## Elysia

The alpha `@sentry/elysia` SDK supports Elysia 1.4+ on Bun and Node.js 18+
through `@elysiajs/node`. It instruments Elysia directly; do not add
`@elysiajs/opentelemetry` for Sentry.

Initialize Sentry before constructing the application. Wrap the application
before registering routes. Its global error hook captures 5xx responses by
default; `shouldHandleError` replaces that policy.

```ts
Sentry.init({ dsn });

const app = Sentry.withElysia(new Elysia(), {
  shouldHandleError: ({ set }) => set.status === 500 || set.status === 503,
}).get("/", () => "ok");
```

## Hono

`@sentry/hono` supports Hono 4+ on Cloudflare Workers, Node.js, Bun, and Deno.
It replaces deprecated `@hono/sentry` and `toucan-js` integrations.

Install the matching same-version peer package:

| Runtime | Peer package | Middleware import |
| --- | --- | --- |
| Cloudflare Workers | `@sentry/cloudflare` | `@sentry/hono/cloudflare` |
| Node.js | `@sentry/node` | `@sentry/hono/node` |
| Bun | `@sentry/bun` | `@sentry/hono/bun` |
| Deno | `@sentry/deno` | `@sentry/hono/deno` |

Add the Sentry middleware early. Cloudflare also requires `nodejs_compat` and
can derive options from Worker bindings:

```ts
import { sentry } from "@sentry/hono/cloudflare";

const app = new Hono<{ Bindings: { SENTRY_DSN: string } }>();
app.use(sentry(app, (env) => ({ dsn: env.SENTRY_DSN })));
```

On Node.js, preload a file that imports and initializes `@sentry/hono/node`,
then call `app.use(sentry(app))` without options.

The middleware captures exceptions reaching Hono's `onError`, excluding 3xx
and 4xx responses by default. Use `shouldHandleError` to replace this filter.

## Nitro

The beta `@sentry/nitro` SDK requires Nitro `3.0.260415-beta` or newer and
Node.js 18.19+.

1. Wrap the Nitro configuration with `withSentryConfig`.
2. Initialize the SDK in a root `instrument.mjs`.
3. Preload that file with `--import` during development, preview, and
   production.

```ts
export default withSentryConfig(defineNitroConfig({}), {
  org: "my-org",
  project: "my-project",
  authToken: process.env.SENTRY_AUTH_TOKEN,
});
```

```sh
NODE_OPTIONS='--import ./instrument.mjs' nitro dev
```

The wrapper registers the module, enables Nitro tracing channels, uploads
hidden source maps by default, and deletes them after upload. It respects an
explicit `sourcemap` setting.

The SDK re-exports `@sentry/node`. It captures unhandled route and middleware
errors through Nitro's `error` hook and skips 3xx/4xx `HTTPError` instances.
Set `enableNitroErrorHandler: false` to disable the hook.

Request tracing uses `h3.request` and `srvx.request`, records parameterized
routes and middleware spans, and exposes `sentry-trace` and `baggage` through
`Server-Timing` so browser pageload traces can link to the server trace.

## TanStack Start

The beta `@sentry/tanstackstart-react` SDK targets TanStack Start 1.0 RC.

For the client and build:

1. Import a client initializer first in `src/client.tsx`.
2. Add `tanstackRouterBrowserTracingIntegration(router)` only when
   `!router.isServer`.
3. Put `sentryTanstackStart()` last in the Vite plugin list. It manages
   production source-map upload and tracing middleware.

```ts
export default defineConfig({
  plugins: [
    tanstackStart(),
    sentryTanstackStart({ org, project, authToken }),
  ],
});
```

For the server:

1. Wrap an explicit server entry's fetch handler with `wrapFetchWithSentry`.
2. For complete instrumentation, preload a root `instrument.server.mjs` with
   `--import` and copy it to the deployed build output.
3. Treat a direct import from `src/server.ts` as a limited fallback. It only
   instruments native Node.js APIs and does not work on Cloudflare.
4. Put `sentryGlobalRequestMiddleware` and
   `sentryGlobalFunctionMiddleware` first in their arrays to capture request
   and server-function errors.
5. Capture SSR rendering exceptions manually.

## TanStack Start event tunneling

Set `sentryTanstackStart({ tunnelRoute: true })` to create an opaque
same-origin tunnel route for each development session and production build.
The plugin configures the client automatically, helping events bypass common
ad-blocker and firewall drops without a hand-written tunnel route.
