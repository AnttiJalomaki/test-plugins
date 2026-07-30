# Build, Source Maps, and Serverless Deployments

## Meta-framework source maps

In the v9 migration (`9.0.0-guide`), meta-framework SDKs stop rewriting
explicitly enabled or disabled source-map settings. When source-map generation
is unspecified, the SDK enables maps and deletes them after upload. When maps
are explicitly enabled, it preserves the chosen emission mode and does not
delete them automatically. Use `filesToDeleteAfterUpload` for custom cleanup.

## Next.js

Next.js enables client `hidden-source-map` and server `source-map` builds unless
`sourcemaps.disable` is set. Client maps are deleted after upload by default;
set `sourcemaps.deleteSourcemapsAfterUpload: false` to retain them.
`hideSourceMaps` is removed without a replacement.

The SDK no longer uses the nondeterministic Next.js Build ID as a release
fallback. Set a release explicitly. Replace the discontinued `sentry` property
in Next config with options passed directly to `withSentryConfig`:

```js
export default withSentryConfig(nextConfig, {
  release: { name: "my-release" },
  sourcemaps: { deleteSourcemapsAfterUpload: false },
});
```

Next.js route handlers flush automatically before returning in serverless
environments (`10.0.0`).

## SolidStart and Remix builds

Replace removed `sentrySolidStartVite` with `withSentry`, passing build-time
Sentry options as its second argument.

Remix always applies automatic instrumentation; `autoInstrumentRemix` is
removed. Sentry CLI failures during Remix source-map upload are silent instead
of failing the build.

## Bundler plugins and diagnostics

Sentry SDK v10 upgrades its bundler plugins to their v4 major line. Check
direct or pinned plugin dependencies during the upgrade (`10.0.0-guide`).

As of 10.68.0, server instrumentation warns when an instrumented module is
marked external by bundler configuration. Treat the warning as an indication
that instrumentation may not be applied; adjust externalization rules or the
deployment layout.

## AWS Lambda and serverless flushing

The v10 AWS Lambda layer is `SentryNodeServerlessSDKv10` and is unified for ESM
and CommonJS. The v9 layer is `SentryNodeServerlessSDKv9`, while maintained v8
updates remain under `SentryNodeServerlessSDKv8`.

React Router flushes automatically for serverless loaders and actions and for
Vercel request handlers. Next.js route handlers also flush automatically.
Unified serverless-environment detection recognizes Cloud Run, so
serverless-specific behavior applies there without custom detection.

## Cloudflare runtime behavior

Cloudflare Workflow instrumentation accepts non-UUID instance IDs. Durable
Object instrumentation keeps synchronous methods synchronous.

The `@sentry/cloudflare/vite` Orchestrion plugin introduced by `10.68.0` reads
Wrangler configuration, resolves the Sentry options module, wraps the worker
entry with `withSentry`, and automatically instruments Durable Object,
`WorkerEntrypoint`, and Workflow classes.

## Nitro deployment

`@sentry/nitro` uses `withSentryConfig`, a root `instrument.mjs`, and a
`--import` preload for development, preview, and production. Its wrapper
registers the module, enables tracing channels, and by default uploads hidden
source maps and deletes them afterward. An explicit `sourcemap` setting is
respected. See [framework-integrations.md](framework-integrations.md) for the
runtime hooks, version floors, and error policy.

## TanStack Start deployment

Place `sentryTanstackStart()` last in the Vite plugin list. It manages
production source-map upload and tracing middleware. For full server coverage,
preload `instrument.server.mjs` with `--import` and include it in the deployed
build output. Importing it from `src/server.ts` is a limited Node-only fallback
and does not work on Cloudflare.

Use `tunnelRoute: true` to generate and configure an opaque same-origin event
tunnel per development session and production build.
