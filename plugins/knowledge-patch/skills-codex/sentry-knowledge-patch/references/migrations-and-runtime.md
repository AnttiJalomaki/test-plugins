# Migrations and runtime compatibility

Use this reference for dependency upgrades, removed APIs, renamed integrations,
runtime support, session behavior, and low-level extension points.

## Runtime and dependency compatibility (9.0.0-guide)

- SDK packages may contain ES2020. General Node.js support starts at 18.0.0;
  the ESM-only Astro, Nuxt, and SvelteKit SDKs start at Node.js 18.19.1. Deno
  starts at 2.0 and TypeScript at 5.0.4.
- Native browser support starts at Chrome/Edge 80, Safari 14, iOS Safari 14.4,
  Firefox 74, Opera 67, and Samsung Internet 13. Transpile the SDK for older
  targets.
- Support is dropped for Remix 1.x, TanStack Router 1.63.0 and lower,
  SvelteKit 1.x, Ember 3.x and lower, and Prisma 5.x.
- Use Lambda layer `SentryNodeServerlessSDKv9` for v9. The v8 update line uses
  `SentryNodeServerlessSDKv8`.
- Import `@sentry/deno` through npm:

```js
import * as Sentry from "npm:@sentry/deno";
```

## Packages, core APIs, and integrations (9.0.0-guide)

- `@sentry/utils` is no longer published. `@sentry/types` is deprecated. Move
  their remaining exports to `@sentry/core`.
- Remove uses of the metrics API, `getCurrentHub()`, `Hub`, and
  `getCurrentHubShim()`.
- Replace `debugIntegration` with send hooks. Replace
  `sessionTimingIntegration` with explicit event context.
- Rename `processThreadBreadcrumbIntegration` to `childProcessIntegration`.
  Its integration name also changes from `ProcessAndThreadBreadcrumbs` to
  `ChildProcess`; update integration-name filters.
- The `vercelAIIntegration` integration name changes from `vercelAI` to
  `VercelAI`; update name-based filtering.

## Session tracking and selection

`autoSessionTracking` is removed (9.0.0-guide). Use the integration matching
the session kind:

- Browser sessions: `browserSessionIntegration`.
- Server request sessions: `httpIntegration`.
- Node.js process sessions: the default `processSessionIntegration`.
- To disable browser sessions, remove `browserSessionIntegration`.
- To disable server request sessions, configure
  `httpIntegration({ trackIncomingRequestsAsSessions: false })`.

Core always selects the session on the isolation scope (9.0.0). When more than
one scope carries session state, do not assume a current-scope session wins.

## Low-level extensions (9.0.0-guide)

- Include `sampleRand` in custom propagation contexts. Replace `spanId` with
  the optional `propagationSpanId`.
- Enrich requests with `httpRequestToRequestData()` and assign its result to
  `event.request`.
- Replace `generatePropagationContext()` with `generateTraceId()`.
- Replace `BAGGAGE_HEADER_NAME` with the literal header name `"baggage"`.
- In v9 custom clients extend `BaseClient`.
- Replace `IntegrationClass` with `Integration` or `IntegrationFn`.

## Remaining v9 API and type changes (9.0.0)

- Core removes `TransactionNamingScheme`, `arrayify()`, `flatten()`,
  `getDomElement()`, `makeFifoCache()`, `memoBuilder`, `urlEncode()`, the
  deprecated `Request` type, and `validSeverityLevels`.
- React removes `getNumberOfUrlSegments()`.
- Next.js removes `experimental_captureRequestError`.
- `recordDroppedEvent()` no longer accepts an event argument.
- Rename `hasTracingEnabled()` to `hasSpansEnabled()`.
- The `shutdownTimeout` option type moves from core to Node.
- The `Scope` interface is replaced by the `Scope` class.
- React's `ErrorBoundary` changes its `componentStack` type.
- The `nitro-utils` package is dropped.

## Core migration to v10 (10.0.0-guide)

- `BaseClient` is removed. Custom clients must use `Client` directly. This
  supersedes the v9 instruction to extend `BaseClient`.
- The `logger` value and `Logger` type are removed. Use `debug` and
  `SentryDebugLogger`.
- Move `_experiments.enableLogs` and `_experiments.beforeSendLog` to the
  top-level initialization options `enableLogs` and `beforeSendLog`.
- Remove Replay's `_experiments.autoFlushOnFeedback`; feedback now triggers a
  replay flush by default.
- Browser SDKs no longer report First Input Delay. Replace FID processing,
  filters, alerts, and dashboards with Interaction to Next Paint equivalents
  where appropriate.

## Packaging and runtime behavior in v10 (10.0.0)

- Sentry SDK bundler plugins move to their v4 major line. Update pins and
  direct plugin consumers for that major-version boundary.
- Use the unified `SentryNodeServerlessSDKv10` AWS Lambda layer for ESM and
  CommonJS.
- Unified serverless detection recognizes Cloud Run and applies relevant
  serverless behavior automatically.
- Cloudflare Workflow instrumentation accepts non-UUID instance IDs. Durable
  Object instrumentation preserves synchronous methods instead of converting
  them to asynchronous methods.
