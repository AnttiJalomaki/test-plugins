# Frameworks, builds, and serverless runtimes

Use this reference for established framework integrations, source-map build
behavior, framework-specific migration, serverless flushing, and runtime
instrumentation.

## Source-map build behavior (9.0.0-guide)

Meta-framework SDKs preserve an explicitly enabled or disabled source-map
setting. When generation is unspecified, they enable source maps and delete
them after upload. When generation is explicitly enabled, they preserve the
emission mode and do not delete maps automatically. Use
`filesToDeleteAfterUpload` for custom cleanup.

Next.js enables client `hidden-source-map` and server `source-map` unless
`sourcemaps.disable` is set. Client maps are deleted after upload unless
`sourcemaps.deleteSourcemapsAfterUpload` is `false`.

- Remove `hideSourceMaps`; it has no replacement.
- Set an explicit release name. The SDK no longer falls back to the
  nondeterministic Next.js Build ID.
- Replace the discontinued `sentry` property in Next config with options
  passed directly to `withSentryConfig`.

```js
export default withSentryConfig(nextConfig, {
  release: { name: "my-release" },
  sourcemaps: { deleteSourcemapsAfterUpload: false },
});
```

## Prisma and NestJS migration (9.0.0-guide)

The bundled `prismaIntegration` targets Prisma 6 and drops Prisma 5. Prisma 6
does not need the `tracing` preview feature. To instrument another Prisma
version, install its matching `@prisma/instrumentation`, pass a
`PrismaInstrumentation` instance through `prismaInstrumentation`, and retain
`previewFeatures = ["tracing"]` for pre-v6 Prisma when required.

```js
Sentry.init({
  integrations: [
    prismaIntegration({
      prismaInstrumentation: new PrismaInstrumentation(),
    }),
  ],
});
```

The Node SDK's `nestIntegration` and `setupNestErrorHandler` are removed. Move
to `@sentry/nestjs` and make these replacements:

- `@WithSentry` -> `@SentryExceptionCaptured`.
- Generic or GraphQL global filter -> `SentryGlobalFilter`.
- Remove `SentryService` and `SentryTracingInterceptor`.

`SentryGlobalFilter` also handles NestJS WebSocket errors (10.68.0).

## React Router, SolidStart, Vue, and Nuxt migration (9.0.0-guide)

- Replace generic React `wrapUseRoutes` and `wrapCreateBrowserRouter` helpers
  with their explicit `V6` or `V7` variants to match React Router.
- Replace removed `sentrySolidStartVite` with `withSentry`, passing build-time
  Sentry options as the second argument:

```ts
export default defineConfig(withSentry(solidStartConfig, sentryBuildOptions));
```

- Put Vue component tracing under `vueIntegration({ tracingOptions: ... })`,
  including in Nuxt. Update spans require `"update"` in
  `tracingOptions.hooks`.
- Pinia `stateTransformer` receives combined state keyed by store ID.
- Remove `logErrors`; the Vue handler always invokes a user handler or
  rethrows.

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

## SvelteKit and Remix migration (9.0.0-guide)

- Remove SvelteKit `fetchProxyScriptNonce`. Use a CSP script hash or disable
  fetch-proxy script injection.
- Remove Remix `autoInstrumentRemix`; the SDK always behaves as if it were
  `true`.

## Framework controls added in 9.0.0

- SolidStart uses `--import` for server setup by default and adds
  `autoInjectServerSentry`, including the
  `"experimental_dynamic-import"` mode.
- Nuxt adds an `enabled` switch for its Sentry module. Nuxt
  `SourceMapsOptions` adds `silent`, `errorHandler`, and `release`.
- SvelteKit injects the fetch-proxy script only on versions below 2.16.0.

## Framework and serverless changes in 10.0.0

- Astro parameterizes Astro 5 request routes and client routes, constructing
  the parameterized request route at runtime.
- `fastifyIntegration` accepts `shouldHandleError` to select captured errors:

```js
Sentry.init({
  integrations: [
    Sentry.fastifyIntegration({
      shouldHandleError: (error) => shouldReport(error),
    }),
  ],
});
```

- React Router automatically flushes serverless loaders, actions, and Vercel
  request handlers. Next.js route handlers also flush.
- Unified serverless detection recognizes Cloud Run.
- Cloudflare Workflow instrumentation accepts non-UUID instance IDs. Durable
  Object instrumentation leaves synchronous methods synchronous.
- Sentry CLI failures during Remix source-map upload are silent instead of
  failing the build.

## Cloudflare and build diagnostics (10.68.0)

Use `@sentry/cloudflare/vite` for Orchestrion auto-instrumentation. The plugin
reads Wrangler configuration, resolves the Sentry options module, wraps the
worker entry with `withSentry`, and instruments Durable Object,
`WorkerEntrypoint`, and Workflow classes automatically.

Server instrumentation warns when a bundler marks an instrumented module
external. Treat the warning as an instrumentation configuration problem; the
external module may not be instrumented.
