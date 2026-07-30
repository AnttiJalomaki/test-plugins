# Tracing, sampling, and spans

Use this reference for sampler migrations, OpenTelemetry setup, trace
continuation, stream mode, span filtering, and current span attributes.

## Span hooks and custom scopes (9.0.0-guide)

`beforeSendSpan` receives root spans as well as child spans. It cannot drop a
span by returning `null`; control which spans are recorded through integration
configuration.

Outside Node.js, `startSpan({ scope })` clones the supplied scope. Mutations to
the current scope stay inside the callback. Also mutate the original custom
scope when a change must persist:

```js
startSpan({ name: "work", scope: customScope }, () => {
  getCurrentScope().setTag("local", "yes");
  customScope.setTag("persistent", "yes");
});
```

## Sampling migration and additions

For the v9 migration (9.0.0-guide):

- Replace `samplingContext.request` with `normalizedRequest`.
- Remove `transactionContext`; fields such as `name` are top-level sampling
  context fields.
- Replace `enableTracing` with `tracesSampleRate`.
- Node.js no longer calls `tracesSampler` for every span.
- Treat an explicitly `undefined` `tracesSampleRate` as absent, allowing a
  downstream service to decide sampling.

```js
Sentry.init({
  tracesSampler: ({ name, normalizedRequest }) =>
    name === "/health-check" ? 0 : 0.5,
});
```

In 9.0.0, the sampler receives `parentSampleRate` and an
`inheritOrSampleWith` helper for parent-aware decisions.

## OpenTelemetry setup

Replace the removed `addOpenTelemetryInstrumentation()` with the
`openTelemetryInstrumentations` initialization option (9.0.0-guide):

```js
Sentry.init({
  openTelemetryInstrumentations: [new GenericPoolInstrumentation()],
});
```

`skipOpenTelemetrySetup: true` also configures `httpIntegration({ spans:
false })` by default. `registerEsmLoaderHooks` accepts only `boolean` or
`undefined` and defaults to wrapping modules used by OpenTelemetry
instrumentation.

Node-based v10 SDKs require OpenTelemetry 2.x/0.20x dependencies and current
instrumentation releases (10.0.0-guide). If a project cannot use OpenTelemetry
2, keep Sentry SDK v9 or use `@sentry/node-core`, whose OpenTelemetry peer
ranges are wider. Sentry SDK v10 remains compatible with Sentry self-hosted
24.4.2 and newer.

## Trace continuation and AI spans (10.0.0)

- Set `strictTraceContinuation: true` to opt into stricter trace-continuation
  behavior.
- OpenAI instrumentation records tool-call attributes and instruments streamed
  responses in Node.js.
- Rename telemetry queries from `ai.response.object` to
  `gen_ai.response.object`.

## Current span semantics (10.68.0)

- DataLoader spans set `cache.key`; Redis delete operations use
  `cache.remove`.
- Parameterized `http.server` spans carry `http.route`, `url.full`, and
  `url.path`. Core fetch instrumentation also emits `url.full`.
- Use the core `instrumentStateGraph` API for state-graph instrumentation.
- Filter `stackFrameVariables` by variable name when only selected captured
  variables should be retained.
- React Router uses its instrumentation API by default. Do not assume the
  older instrumentation path is automatically selected.

## Stream mode configuration (streamed-spans)

SDK 10.66.0 and newer can opt into stream mode. It sends completed spans in
periodic batches instead of retaining a transaction until its root span ends.
Service spans replace transactions as service entry points, and the
transaction mode's 1,000-span ceiling does not apply. Stream mode becomes the
default in SDK 11. Cordova and Electron do not support it.

Configure server and serverless SDKs directly:

```js
Sentry.init({ tracesSampleRate: 1, traceLifecycle: "stream" });
```

Configure direct browser SDKs with the integration:

```js
Sentry.init({
  tracesSampleRate: 1,
  integrations: [Sentry.spanStreamingIntegration()],
});
```

Tracing mode is per SDK, so transaction-mode and stream-mode services can join
the same distributed trace. Completed spans flush every five seconds by
default, when 1,000 spans are buffered, when the batch-size limit is reached,
or on `Sentry.flush()` and `Sentry.close()`.

## Stream-aware filtering

Wrap `beforeSendSpan` with `Sentry.withStreamedSpan()`. An unwrapped hook causes
fallback to transaction mode.

```js
Sentry.init({
  traceLifecycle: "stream",
  beforeSendSpan: Sentry.withStreamedSpan((span) => {
    if (span.attributes?.["sentry.op"] === "db.query") {
      span.name = "[filtered]";
    }
    return span;
  }),
});
```

The hook receives `StreamedSpanJSON`. Map fields as follows:

| Transaction representation | Streamed representation |
| --- | --- |
| `description` | `name` |
| processed `data` | raw `attributes` |
| `timestamp` | `end_timestamp` |
| `op` | `attributes["sentry.op"]` |
| richer status | `"ok"` or `"error"` |

`beforeSendTransaction` is unavailable. Move transaction-dropping rules to
`ignoreSpans`.

## Start-time dropping and attributes

`ignoreSpans` evaluates the initial span name and attributes when the span
starts. Later renaming or enrichment cannot affect the decision. Rules accept
name strings, regular expressions, or name-and-attribute objects:

```js
Sentry.init({
  traceLifecycle: "stream",
  ignoreSpans: [
    "healthcheck",
    /^GET \/api\/v1\/internal/,
    { name: /^GET \//, attributes: { "http.route": "/api/status" } },
  ],
});
```

Dropping a service span drops its descendants. Dropping a child reparents its
children to the nearest retained ancestor.

Stream mode does not copy `Sentry.setTag()` or `Sentry.setTags()` values to
spans, though tags still apply to errors. Use attribute APIs for span data.
