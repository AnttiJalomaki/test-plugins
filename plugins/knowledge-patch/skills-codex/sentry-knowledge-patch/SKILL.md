---
name: sentry-knowledge-patch
description: Sentry JavaScript SDK
version: 10.68.0
license: MIT
metadata:
  author: Nevaberry
---


# Sentry JavaScript SDK

Use this skill when configuring, upgrading, or debugging Sentry JavaScript SDKs.
Start with the installed `@sentry/*` package versions and the application's
runtime and framework versions. Apply version-dependent guidance only when it
matches the project, and prefer the manifest, code, tests, and observed runtime
behavior when they disagree with assumptions.

## Reference index

| Reference | Topics |
| --- | --- |
| [migrations-and-runtime.md](references/migrations-and-runtime.md) | Runtime floors, package and API removals, sessions, renamed integrations, low-level APIs |
| [tracing-and-spans.md](references/tracing-and-spans.md) | Sampling, OpenTelemetry, span hooks, strict continuation, span streaming, telemetry attributes |
| [logging-privacy-and-events.md](references/logging-privacy-and-events.md) | Structured logs, console ingestion, shared attributes, PII, feedback, event enrichment |
| [frameworks-and-builds.md](references/frameworks-and-builds.md) | Next.js, Remix, SvelteKit, Nuxt, Vue, Astro, NestJS, Prisma, serverless, source maps |
| [modern-server-frameworks.md](references/modern-server-frameworks.md) | Elysia, Hono, Nitro, TanStack Start, request controls, middleware ordering, tunneling |

## Upgrade workflow

1. Inventory every direct `@sentry/*` dependency, bundler plugin, Lambda
   layer, preload file, and framework wrapper.
2. Confirm Node.js, browser, Deno, TypeScript, and framework support before
   changing packages.
3. Remove deleted options and APIs before debugging new behavior. Do not keep
   obsolete options as inert compatibility shims.
4. Update integration factory names and any filters keyed by integration
   `name` values.
5. Recheck sampling, session tracking, source-map generation, upload, and
   deletion independently; they no longer share all former defaults.
6. Audit user data, request bodies, IP inference, console capture, logs, and
   replay behavior after migration.
7. Exercise errors, traces, logs, source maps, flush behavior, and graceful
   shutdown in the actual deployment runtime.

## Breaking-change quick reference

### Runtime and package floors

- Expect ES2020 in SDK packages.
- Require Node.js 18.0.0 generally; require Node.js 18.19.1 for the ESM-only
  Astro, Nuxt, and SvelteKit SDKs.
- Require Deno 2.0 and TypeScript 5.0.4.
- Transpile the SDK for browsers older than Chrome/Edge 80, Safari 14, iOS
  Safari 14.4, Firefox 74, Opera 67, or Samsung Internet 13.
- Do not combine current SDKs with Remix 1.x, TanStack Router 1.63.0 or older,
  SvelteKit 1.x, Ember 3.x or older, or Prisma 5.x.

### Migrate to v9

- Replace `enableTracing` with `tracesSampleRate`; replace
  `samplingContext.request` with `normalizedRequest` and read former
  `transactionContext` fields directly from the sampling context.
- Replace `autoSessionTracking` with `browserSessionIntegration`,
  `httpIntegration`, or the default `processSessionIntegration`, depending on
  the session type.
- Import surviving `@sentry/utils` and deprecated `@sentry/types` exports from
  `@sentry/core`; remove dependencies on the metrics API and Hub APIs.
- Replace `captureUserFeedback()` with `captureFeedback()` and rename payload
  `comments` to `message`.
- Replace `hasTracingEnabled()` with `hasSpansEnabled()` and treat `Scope` as a
  class rather than the removed interface.
- Import Deno from `npm:@sentry/deno`, not `deno.land`.

### Migrate to v10

- Upgrade directly consumed or pinned Sentry bundler plugins to their v4
  major line.
- Move `_experiments.enableLogs` and `_experiments.beforeSendLog` to top-level
  initialization options.
- Replace custom `BaseClient` use with `Client`; replace `logger` and `Logger`
  with `debug` and `SentryDebugLogger`.
- Upgrade Node OpenTelemetry dependencies to the 2.x/0.20x line, remain on
  Sentry v9 when that is impossible, or evaluate `@sentry/node-core` for its
  wider OpenTelemetry peer ranges.
- Remove FID-dependent processing and use INP-oriented processing where
  appropriate.
- Use the unified `SentryNodeServerlessSDKv10` Lambda layer for ESM and
  CommonJS deployments.

## Tracing quick reference

### Sampling

```js
Sentry.init({
  tracesSampler: ({ name, normalizedRequest, parentSampleRate, inheritOrSampleWith }) =>
    name === "/health-check" ? 0 : inheritOrSampleWith(0.5),
});
```

An explicitly `undefined` `tracesSampleRate` is absent, allowing a downstream
service to decide. Node.js does not invoke `tracesSampler` for every child
span. Use `strictTraceContinuation: true` only when stricter incoming trace
continuation is intended.

### Stream spans

For SDK 10.66.0 and newer, opt a server SDK into periodic span delivery:

```js
Sentry.init({
  tracesSampleRate: 1,
  traceLifecycle: "stream",
  beforeSendSpan: Sentry.withStreamedSpan((span) => span),
});
```

Use `spanStreamingIntegration()` for direct browser SDKs. Keep
`beforeSendSpan` wrapped with `withStreamedSpan`; an unwrapped hook falls back
to transaction mode. Replace transaction-dropping rules with `ignoreSpans`,
which evaluates only initial names and attributes when each span starts.

Do not expect `setTag()` or `setTags()` values on streamed spans. Record span
data with attributes while retaining tags when errors also need them.

## Logs quick reference

Enable logs at the top level, emit a required message, and attach searchable
attributes as the second argument:

```js
Sentry.init({
  enableLogs: true,
  integrations: [
    Sentry.consoleLoggingIntegration({ levels: ["log", "warn", "error"] }),
  ],
});

Sentry.setAttributes({ org_id: org.id, user_tier: user.tier });
Sentry.logger.info("Order created", { orderId: "order_456" });
```

Use `Sentry.logger.fmt` when interpolated values should become searchable
attributes. Logs within active spans or supported browser replays gain trace
or replay correlation automatically.

## Privacy and event quick reference

Browser SDKs do not request backend IP inference by default. Enable it only
deliberately:

```js
Sentry.init({
  sendDefaultPii: true,
  dataCollection: { userInfo: false, httpBodies: [] },
});
```

`requestDataIntegration` does not infer an Express user from `request.user`;
call `Sentry.setUser()` explicitly. For Elysia, Hono, Nitro, and TanStack Start,
use `dataCollection` to turn off default user information and request-body
capture.

## Framework selection

| Work area | Read first |
| --- | --- |
| Next.js, Remix, Astro, Vue, Nuxt, SvelteKit, SolidStart | [frameworks-and-builds.md](references/frameworks-and-builds.md) |
| NestJS, Prisma, React Router, Fastify | [frameworks-and-builds.md](references/frameworks-and-builds.md) |
| AWS Lambda, Cloud Run, Cloudflare runtime behavior | [frameworks-and-builds.md](references/frameworks-and-builds.md) |
| Elysia 1.4+ | [modern-server-frameworks.md](references/modern-server-frameworks.md#elysia) |
| Hono 4+ | [modern-server-frameworks.md](references/modern-server-frameworks.md#hono) |
| Nitro 3 beta | [modern-server-frameworks.md](references/modern-server-frameworks.md#nitro) |
| TanStack Start 1.0 RC | [modern-server-frameworks.md](references/modern-server-frameworks.md#tanstack-start) |

## Verification checklist

- Run the application in each deployed module format and runtime.
- Verify the correct release identifier; do not rely on a Next.js Build ID.
- Confirm source maps are generated, uploaded, and retained or deleted as
  intended.
- Capture a handled error, an unhandled framework error, and an error excluded
  by `shouldHandleError` where configured.
- Verify parent sampling, distributed trace continuation, and service-span
  behavior across mixed tracing modes.
- Flush or close the SDK in short-lived and serverless processes.
- Inspect emitted log attributes, trace correlation, replay correlation, HTTP
  route attributes, and any user or body data.
- Check bundler warnings for instrumented modules marked external.
