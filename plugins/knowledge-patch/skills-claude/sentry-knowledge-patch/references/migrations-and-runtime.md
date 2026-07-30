# Migrations and Runtime Compatibility

## Runtime and dependency floors

For the v9 migration (`9.0.0-guide`), SDK packages may emit ES2020. The general
Node.js floor is 18.0.0; the ESM-only Astro, Nuxt, and SvelteKit SDKs require
18.19.1. The other floors are Deno 2.0 and TypeScript 5.0.4.

Native browser floors are Chrome and Edge 80, Safari 14, iOS Safari 14.4,
Firefox 74, Opera 67, and Samsung Internet 13. Transpile the SDK for older
targets.

Support is dropped for Remix 1.x, TanStack Router 1.63.0 and lower, SvelteKit
1.x, Ember 3.x and lower, and Prisma 5.x. Prisma-specific migration details
are in [framework-integrations.md](framework-integrations.md).

## Package and integration moves

- `@sentry/utils` is no longer published. `@sentry/types` is deprecated. Move
  their remaining exports to `@sentry/core`.
- `processThreadBreadcrumbIntegration` becomes `childProcessIntegration`, and
  the integration name becomes `ChildProcess` instead of
  `ProcessAndThreadBreadcrumbs`.
- `vercelAIIntegration` keeps its factory spelling, but its integration name
  changes from `vercelAI` to `VercelAI`. Update integration-name filters.
- `@sentry/deno` is no longer published on `deno.land`:

```js
import * as Sentry from "npm:@sentry/deno";
```

## Core API removals

The v9 release (`9.0.0`) removes these core exports:

- `TransactionNamingScheme`, `arrayify()`, `flatten()`, `getDomElement()`,
  `makeFifoCache()`, `memoBuilder`, `urlEncode()`, the deprecated `Request`
  type, and `validSeverityLevels`;
- the metrics API, `getCurrentHub()`, `Hub`, and `getCurrentHubShim()`;
- React's `getNumberOfUrlSegments()`;
- Next.js's `experimental_captureRequestError`;
- the `nitro-utils` package.

Further signature and type changes:

- `recordDroppedEvent()` no longer accepts an event argument.
- `hasTracingEnabled()` is renamed to `hasSpansEnabled()`.
- The `shutdownTimeout` option type moves from core to Node.
- The `Scope` type interface is replaced by the `Scope` class.
- React `ErrorBoundary` changes the type of `componentStack`.
- Replace `debugIntegration` with send hooks.
- Replace `sessionTimingIntegration` with explicit event context.

## Low-level extension APIs

- Custom propagation contexts require `sampleRand`.
- Replace `spanId` with optional `propagationSpanId`.
- Enrich requests with `httpRequestToRequestData()` and assign the result to
  `event.request`.
- Replace `generatePropagationContext()` with `generateTraceId()`.
- Replace `BAGGAGE_HEADER_NAME` with the literal `"baggage"`.
- Replace `IntegrationClass` with `Integration` or `IntegrationFn`.

During the v9 migration, custom clients had to extend `BaseClient`. At the v10
boundary (`10.0.0-guide`), `BaseClient` is removed, so v10 custom clients use
`Client` directly. This is a versioned transition, not interchangeable advice.

V10 also removes the `logger` value and `Logger` type. Use `debug` and
`SentryDebugLogger` instead.

## Feedback and browser API changes

`captureUserFeedback()` is removed. Use `captureFeedback()` and rename the
payload's `comments` field to `message`:

```js
Sentry.captureFeedback({ message: "What happened" });
```

Browser SDKs no longer report First Input Delay in v10. Replace FID-dependent
processing, filters, alerts, and dashboards with Interaction to Next Paint
equivalents where appropriate.

## AWS Lambda layer names

- The v9 layer is `SentryNodeServerlessSDKv9`; continuing v8 updates use
  `SentryNodeServerlessSDKv8`.
- The v10 layer is `SentryNodeServerlessSDKv10` and is unified for ESM and
  CommonJS deployments (`10.0.0`).
