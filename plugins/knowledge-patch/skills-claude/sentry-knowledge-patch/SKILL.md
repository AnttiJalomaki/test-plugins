---
name: sentry-knowledge-patch
description: Sentry JavaScript SDK
version: 10.68.0
license: MIT
metadata:
  author: Nevaberry
---


# Sentry JavaScript SDK Knowledge Patch

Use this skill when upgrading, configuring, or debugging Sentry's JavaScript
and TypeScript SDKs. Check the installed SDK and framework versions before
applying version-specific advice. Prefer the application's manifests, code,
and tests when they disagree with an example.

## Reference index

| Reference | Topics |
| --- | --- |
| [migrations-and-runtime.md](references/migrations-and-runtime.md) | Runtime floors, removed packages and APIs, renamed integrations, client changes |
| [tracing-and-performance.md](references/tracing-and-performance.md) | Sampling, OpenTelemetry, span hooks, stream mode, span attributes |
| [telemetry-data.md](references/telemetry-data.md) | Sessions, privacy, feedback, structured logs, shared attributes, variable capture |
| [framework-integrations.md](references/framework-integrations.md) | Prisma, NestJS, React Router, Vue, Nuxt, SvelteKit, Elysia, Hono, Nitro, TanStack Start |
| [build-and-serverless.md](references/build-and-serverless.md) | Source maps, bundlers, Lambda, Cloudflare, serverless flushing and deployment |

## Upgrade triage

1. Read the installed `@sentry/*` versions and the runtime/framework versions.
2. Raise runtime floors before changing SDK APIs.
3. Replace removed imports, options, integrations, and wrapper functions.
4. Verify tracing and session behavior; several old options no longer control it.
5. Recheck source-map generation, upload, and deletion in production builds.
6. Run framework-specific error, trace, log, and shutdown-flush tests.

## Breaking changes and deprecations

### Runtime floors

- SDK packages may contain ES2020.
- Use Node.js 18.0.0 or newer. Astro, Nuxt, and SvelteKit require Node.js
  18.19.1 because their SDKs are ESM-only.
- Use Deno 2.0 and TypeScript 5.0.4 or newer.
- For older browser targets, transpile the SDK. Native floors include
  Chrome/Edge 80, Safari 14, iOS Safari 14.4, Firefox 74, Opera 67, and
  Samsung Internet 13.
- Do not expect support for Remix 1.x, SvelteKit 1.x, Ember 3.x or lower,
  TanStack Router 1.63.0 or lower, or Prisma 5.x.

### Sampling and span hooks

- Replace `samplingContext.request` with `normalizedRequest`.
- Read transaction fields such as `name` directly from the sampling context;
  `transactionContext` is gone.
- Replace `enableTracing` with a defined `tracesSampleRate` or a
  `tracesSampler`. An explicitly `undefined` sample rate is treated as absent.
- In Node.js, `tracesSampler` is not invoked for every child span.
- `beforeSendSpan` sees root and child spans and cannot drop spans by returning
  `null`. Configure recording or use `ignoreSpans` where supported.
- Outside Node.js, `startSpan({ scope })` clones the supplied scope. Mutate the
  original scope too when changes must persist after the callback.

```js
Sentry.init({
  tracesSampler: ({ name, normalizedRequest, inheritOrSampleWith }) =>
    name === "/health-check" ? 0 : inheritOrSampleWith(0.5),
});
```

### Sessions, privacy, and event data

- Remove `autoSessionTracking`. Browser sessions come from
  `browserSessionIntegration`, incoming server sessions from `httpIntegration`,
  and Node.js process sessions from `processSessionIntegration`.
- To stop browser session tracking, remove its integration. To stop incoming
  request sessions, use
  `httpIntegration({ trackIncomingRequestsAsSessions: false })`.
- Browser SDKs do not request backend IP inference by default. Enable
  `sendDefaultPii` only when that data collection is intended.
- `requestDataIntegration` no longer copies Express `request.user`; call
  `Sentry.setUser()` explicitly in middleware.
- Replace `captureUserFeedback({ comments })` with
  `captureFeedback({ message })`.

### Packages, clients, and removed APIs

- Move surviving `@sentry/utils` and `@sentry/types` imports to
  `@sentry/core`; utils is unpublished and types is deprecated.
- Remove uses of the old metrics API, hubs, and hub shims.
- Replace `hasTracingEnabled()` with `hasSpansEnabled()`.
- Replace `debugIntegration` with send hooks and
  `sessionTimingIntegration` with explicit event context.
- For v9 custom clients, the migration target was `BaseClient`; in v10,
  `BaseClient` itself is removed and custom clients use `Client` directly.
- In v10, replace the removed `logger` value and `Logger` type with `debug`
  and `SentryDebugLogger`.

### Integration and framework renames

- Rename `processThreadBreadcrumbIntegration` to `childProcessIntegration`;
  integration-name filters must use `ChildProcess`.
- Update Vercel AI integration-name filters from `vercelAI` to `VercelAI`.
- Use `@sentry/nestjs` instead of Node's removed Nest integration and setup
  helper. Use `SentryExceptionCaptured` and `SentryGlobalFilter`.
- Choose explicit React Router `V6` or `V7` route wrappers.
- Use `withSentry` instead of `sentrySolidStartVite`.
- Import Deno from `npm:@sentry/deno`, not `deno.land`.

## Build and deployment quick reference

### Source maps

- Meta-framework SDKs preserve explicit source-map settings. When generation
  is unspecified, they generate maps and delete them after upload.
- Explicitly enabled source maps are not deleted automatically; configure
  `filesToDeleteAfterUpload` when cleanup is required.
- Next.js defaults to client `hidden-source-map` and server `source-map`
  unless `sourcemaps.disable` is set. Use
  `sourcemaps.deleteSourcemapsAfterUpload` to retain client maps.
- Pass SDK build options directly to `withSentryConfig`; the old Next.js
  `sentry` config property and `hideSourceMaps` are removed.
- Set a deterministic release explicitly; the SDK no longer falls back to
  the Next.js Build ID.

### Serverless behavior

- The v10 Lambda layer is `SentryNodeServerlessSDKv10` and supports ESM and
  CommonJS. The v9 layer is `SentryNodeServerlessSDKv9`.
- React Router serverless loaders/actions, Vercel handlers, and Next.js route
  handlers flush automatically.
- Unified serverless detection recognizes Cloud Run.
- Sentry CLI source-map upload failures in Remix are silent.

## High-value capabilities

### Structured logs

Enable logs at the top level and use structured attributes for searchable
context. The experimental option names are obsolete.

```js
Sentry.init({ enableLogs: true, beforeSendLog: log => log });
Sentry.logger.info("Order created", { orderId: "order_456" });
Sentry.logger.info(Sentry.logger.fmt`User ${userId} purchased ${productName}`);
```

Use `setAttribute` or `setAttributes` for values shared by logs and metrics.
Scope methods provide app-wide or operation-local placement. Active spans and
supported browser replays automatically correlate emitted logs.

### Streamed spans

SDK 10.66.0 and newer can opt into stream mode. Server SDKs use
`traceLifecycle: "stream"`; direct browser SDKs add
`spanStreamingIntegration()`. Cordova and Electron do not support it.

```js
Sentry.init({
  tracesSampleRate: 1,
  traceLifecycle: "stream",
  beforeSendSpan: Sentry.withStreamedSpan(span => span),
});
```

- Wrap `beforeSendSpan` with `withStreamedSpan`; an unwrapped hook forces a
  fallback to transaction mode.
- `beforeSendTransaction` is unavailable. Move dropping rules to
  `ignoreSpans`, which evaluates initial names and attributes at span start.
- Span tags are not propagated in stream mode. Store span data as attributes.
- Completed spans flush periodically, at capacity, and on `flush()` or
  `close()`; a long root span no longer retains all children.

### Current tracing controls

- Set `strictTraceContinuation: true` to opt into strict continuation.
- Use `fastifyIntegration({ shouldHandleError })` to select captured errors.
- Streamed responses and tool calls from the instrumented AI client are
  captured; query
  `gen_ai.response.object`, not the old `ai.response.object` attribute.
- Parameterized server spans include `http.route`, `url.full`, and `url.path`;
  fetch spans include `url.full`.
- `instrumentStateGraph` is the supported state-graph instrumentation API.

### Modern server frameworks

- Elysia, Hono, Nitro, and TanStack Start SDKs accept
  `dataCollection: { userInfo: false, httpBodies: [] }` when default user and
  request-body collection must be disabled.
- Initialize or preload Sentry before application construction and framework
  imports where required.
- Put Sentry middleware early so exceptions reach it; keep build plugins in
  the framework-specific required position.
- Use each integration's `shouldHandleError` or error-handler control to make
  capture policy explicit.

Consult the framework and build references before copying setup snippets;
runtime entry points, preload requirements, source-map behavior, and default
error filters differ substantially.
