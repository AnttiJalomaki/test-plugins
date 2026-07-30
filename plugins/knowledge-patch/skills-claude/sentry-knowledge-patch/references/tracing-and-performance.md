# Tracing and Performance

## Sampling context and configuration

The v9 migration removes `samplingContext.request`; use `normalizedRequest`.
It also removes `transactionContext`, placing fields such as `name` directly
on the sampling context. Replace `enableTracing` with `tracesSampleRate`.
Explicitly setting `tracesSampleRate: undefined` is treated as though the
option were absent, allowing a downstream service to decide.

Node.js no longer calls `tracesSampler` for every span. Since `9.0.0`, the
sampler also receives `parentSampleRate` and the `inheritOrSampleWith` helper:

```js
Sentry.init({
  tracesSampler: ({ name, normalizedRequest, parentSampleRate,
                    inheritOrSampleWith }) =>
    name === "/health-check" ? 0 : inheritOrSampleWith(0.5),
});
```

## Span hooks and scope behavior

`beforeSendSpan` receives root spans as well as child spans. It cannot drop a
span by returning `null`; control recording through integration configuration,
or use `ignoreSpans` in stream mode.

Outside Node.js, `startSpan({ scope })` clones the supplied scope. Current-scope
mutations stay inside the callback. Mutate the original scope separately when
the change must persist:

```js
startSpan({ name: "work", scope: customScope }, () => {
  getCurrentScope().setTag("local", "yes");
  customScope.setTag("persistent", "yes");
});
```

## OpenTelemetry compatibility and setup

`addOpenTelemetryInstrumentation()` is removed. Provide custom instrumentation
through `openTelemetryInstrumentations`:

```js
Sentry.init({
  openTelemetryInstrumentations: [new GenericPoolInstrumentation()],
});
```

`skipOpenTelemetrySetup: true` now also configures
`httpIntegration({ spans: false })` by default. `registerEsmLoaderHooks`
accepts only a boolean or `undefined` and defaults to wrapping modules used by
OpenTelemetry instrumentation.

Node-based v10 SDKs require the OpenTelemetry 2.x/0.20x dependency generation
and current instrumentations (`10.0.0-guide`). If the application cannot use
OpenTelemetry 2, stay on Sentry v9 or use `@sentry/node-core`, whose peer
dependency ranges are broader. V10 remains compatible with self-hosted Sentry
24.4.2 and newer.

## Trace continuation and instrumentation

- Enable stricter continuation rules with
  `Sentry.init({ strictTraceContinuation: true })`.
- The AI client instrumentation captures tool-call attributes and streamed
  responses in Node.js. Rename telemetry queries from `ai.response.object` to
  `gen_ai.response.object`.
- DataLoader spans set `cache.key`; Redis delete operations use
  `cache.remove` (`10.68.0`).
- Parameterized `http.server` spans carry `http.route`, `url.full`, and
  `url.path`. Core fetch instrumentation also adds `url.full`.
- Use the core `instrumentStateGraph` API for supported state-graph
  instrumentation.

## Stream mode

The `streamed-spans` capability is opt-in from SDK 10.66.0 and becomes the
default in SDK 11. It sends completed spans in periodic batches rather than
holding an entire transaction until the root ends. Service spans replace
transactions as service entry points, and the transaction mode's 1,000-span
ceiling no longer applies. Cordova and Electron do not support stream mode.

```js
// Server and serverless SDKs
Sentry.init({ tracesSampleRate: 1, traceLifecycle: "stream" });

// Direct browser SDKs
Sentry.init({
  tracesSampleRate: 1,
  integrations: [Sentry.spanStreamingIntegration()],
});
```

Mode is scoped to each SDK, so transaction-mode and stream-mode services can
share a distributed trace. Completed spans flush every five seconds by
default, when 1,000 spans are buffered, at the batch-size limit, and on
`Sentry.flush()` or `Sentry.close()`.

### Filtering streamed spans

Wrap `beforeSendSpan` with `Sentry.withStreamedSpan()`. An unwrapped hook makes
the SDK fall back to transaction mode.

```js
Sentry.init({
  traceLifecycle: "stream",
  beforeSendSpan: Sentry.withStreamedSpan(span => {
    if (span.attributes?.["sentry.op"] === "db.query") {
      span.name = "[filtered]";
    }
    return span;
  }),
});
```

The hook receives `StreamedSpanJSON`. Its shape differs from transaction-mode
span JSON:

| Previous field | Streamed field |
| --- | --- |
| `description` | `name` |
| processed `data` | raw `attributes` |
| `timestamp` | `end_timestamp` |
| `op` | `attributes["sentry.op"]` |

Status is `"ok"` or `"error"`. `beforeSendTransaction` is unavailable; move
transaction-dropping rules to `ignoreSpans`.

`ignoreSpans` evaluates only the initial name and attributes at span start.
Later renaming or enrichment cannot change the decision. Rules can be name
strings, regular expressions, or name-and-attribute objects:

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

## Attributes in stream mode

`Sentry.setTag()` and `Sentry.setTags()` still affect errors but no longer add
values to streamed spans. Record span-relevant data with attribute APIs.
