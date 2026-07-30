# Framework Integrations

## Prisma

The bundled `prismaIntegration` targets Prisma 6 and no longer supports Prisma
5. Prisma 6 does not need the `tracing` preview feature. To instrument another
version, install its matching `@prisma/instrumentation`, pass a
`PrismaInstrumentation` instance, and keep `previewFeatures = ["tracing"]`
for pre-v6 Prisma when required.

```js
Sentry.init({
  integrations: [
    Sentry.prismaIntegration({
      prismaInstrumentation: new PrismaInstrumentation(),
    }),
  ],
});
```

## NestJS

The v9 migration removes the Node SDK's `nestIntegration` and
`setupNestErrorHandler`. Use `@sentry/nestjs` instead:

- replace `@WithSentry` with `@SentryExceptionCaptured`;
- replace generic and GraphQL exception filters with `SentryGlobalFilter`;
- remove `SentryService` and `SentryTracingInterceptor`.

As of 10.68.0, `SentryGlobalFilter` also handles WebSocket errors.

## React Router

The generic `wrapUseRoutes` and `wrapCreateBrowserRouter` helpers are removed.
Choose the explicit `V6` or `V7` variant matching the installed React Router
major. React Router now uses its instrumentation API by default; current custom
setup must not assume the older instrumentation path is auto-selected.

React Router automatically flushes for serverless loaders and actions and for
Vercel request handlers. This behavior arrived at the v10 boundary
(`10.0.0`).

## SolidStart

`sentrySolidStartVite` is removed. Wrap the SolidStart config with `withSentry`
and provide build-time options as the second argument:

```ts
export default defineConfig(
  withSentry(solidStartConfig, sentryBuildOptions),
);
```

SolidStart defaults server setup to `--import`. It supports
`autoInjectServerSentry`, including
`autoInjectServerSentry: "experimental_dynamic-import"`.

## Vue and Nuxt

Configure Vue component tracing under `vueIntegration({ tracingOptions })`,
including in Nuxt. Update spans are opt-in; include `"update"` in `hooks`:

```js
Sentry.init({
  integrations: [
    Sentry.vueIntegration({
      tracingOptions: {
        trackComponents: true,
        hooks: ["mount", "update", "unmount"],
      },
    }),
  ],
});
```

Pinia `stateTransformer` receives combined state keyed by store ID. The
removed `logErrors` option is unnecessary because the Vue handler always
propagates to a user handler or rethrows.

Nuxt's module has an `enabled` switch. Its `SourceMapsOptions` includes
`silent`, `errorHandler`, and `release`.

## SvelteKit and Remix

SvelteKit removes `fetchProxyScriptNonce`; use a CSP script hash or disable
fetch-proxy injection. It injects that script only for SvelteKit versions below
2.16.0. Remix removes `autoInstrumentRemix` and always behaves as if it were
enabled.

## Astro and Fastify

Astro 5 request routes and client-side routes are parameterized, with the
request route constructed at runtime. Fastify exposes error selection through
`shouldHandleError`:

```js
Sentry.init({
  integrations: [
    Sentry.fastifyIntegration({
      shouldHandleError: error => shouldReport(error),
    }),
  ],
});
```

## Statsig and worker threads

The browser SDK includes a Statsig integration. The Node SDK captures
exceptions from `worker_threads` (both since the v9 line).

## Cloudflare runtime classes

Cloudflare Workflow instrumentation accepts non-UUID instance IDs. Durable
Object instrumentation preserves synchronous methods instead of making them
asynchronous.

## Shared modern-framework controls

The `modern-server-frameworks` SDKs for Elysia, Hono, Nitro, and TanStack Start
accept `dataCollection` controls. Disable automatic user data and all request
body collection explicitly when required:

```js
Sentry.init({ dataCollection: { userInfo: false, httpBodies: [] } });
```

## Elysia

The alpha `@sentry/elysia` SDK supports Elysia 1.4+ on Bun and Node.js 18+
through `@elysiajs/node`. It instruments Elysia natively; do not add
`@elysiajs/opentelemetry`. Initialize Sentry before constructing the app and
wrap the app before registering routes.

```ts
Sentry.init({ dsn });
const app = Sentry.withElysia(new Elysia(), {
  shouldHandleError: ({ set }) => set.status === 500 || set.status === 503,
}).get("/", () => "ok");
```

The global error hook captures 5xx responses by default. `shouldHandleError`
replaces that policy.

## Hono

`@sentry/hono` supports Hono 4+ on Cloudflare Workers, Node.js, Bun, and Deno.
It replaces deprecated `@hono/sentry` and `toucan-js`. Install the same-version
peer for the runtime: `@sentry/cloudflare`, `@sentry/node`, `@sentry/bun`, or
`@sentry/deno`. Import `sentry` from `@sentry/hono/<runtime>` and register it
early.

```ts
import { sentry } from "@sentry/hono/cloudflare";

const app = new Hono<{ Bindings: { SENTRY_DSN: string } }>();
app.use(sentry(app, env => ({ dsn: env.SENTRY_DSN })));
```

Cloudflare additionally requires `nodejs_compat` and may derive options from
Worker bindings. On Node.js, preload a file that imports and initializes
`@sentry/hono/node`, then call `app.use(sentry(app))` without options.

The middleware captures exceptions reaching Hono's `onError`, excluding 3xx
and 4xx by default. Use `shouldHandleError` to change that selection.

## Nitro

The beta `@sentry/nitro` SDK requires Nitro `3.0.260415-beta` or newer and
Node.js 18.19+. Wrap config with `withSentryConfig`, initialize in a root
`instrument.mjs`, and preload it with `--import` in development, preview, and
production:

```ts
export default withSentryConfig(defineNitroConfig({}), {
  org: "my-org",
  project: "my-project",
  authToken: process.env.SENTRY_AUTH_TOKEN,
});
// NODE_OPTIONS='--import ./instrument.mjs' nitro dev
```

The wrapper registers the module, enables Nitro tracing channels, uploads
hidden source maps, and deletes them by default while respecting explicit
`sourcemap` settings. The SDK re-exports `@sentry/node`.

Nitro's error hook captures unhandled route and middleware errors but skips
3xx/4xx `HTTPError`s. Set `enableNitroErrorHandler: false` to disable it.
Tracing uses `h3.request` and `srvx.request`, captures parameterized routes and
middleware spans, and publishes `sentry-trace` and `baggage` via
`Server-Timing` so browser pageloads can connect to the server trace.

## TanStack Start

The beta `@sentry/tanstackstart-react` SDK targets TanStack Start 1.0 RC.

- Import a client initializer first in `src/client.tsx`.
- Add `tanstackRouterBrowserTracingIntegration(router)` only when
  `!router.isServer`.
- Put `sentryTanstackStart()` last in the Vite plugin list.
- Wrap an explicit server entry fetch handler with `wrapFetchWithSentry`.

```ts
export default defineConfig({
  plugins: [
    tanstackStart(),
    sentryTanstackStart({ org, project, authToken }),
  ],
});
```

The Vite plugin manages production source-map upload and tracing middleware.
For complete server instrumentation, preload a root `instrument.server.mjs`
with `--import` and copy it into the deployment output. Directly importing it
from `src/server.ts` instruments only native Node.js APIs and does not work on
Cloudflare.

Place `sentryGlobalRequestMiddleware` and
`sentryGlobalFunctionMiddleware` first in their arrays to capture request and
server-function errors. Capture SSR rendering exceptions manually.

Set `sentryTanstackStart({ tunnelRoute: true })` to create an opaque same-origin
tunnel route per development session and production build. The plugin
configures the client automatically, reducing ad-blocker and firewall drops.
