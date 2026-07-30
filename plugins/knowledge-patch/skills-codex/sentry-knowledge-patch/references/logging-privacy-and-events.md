# Logging, privacy, and events

Use this reference for structured logging, shared log and metric attributes,
console ingestion, replay and trace correlation, PII controls, feedback, and
manual event enrichment.

## Browser and console event behavior (9.0.0-guide)

Browser SDKs no longer request backend IP inference by default. Set
`sendDefaultPii: true` only when IP inference is intended.

With `attachStackTrace: true`, `captureConsoleIntegration` marks console events
handled unless configured with `{ handled: false }`:

```js
Sentry.init({
  sendDefaultPii: true,
  attachStackTrace: true,
  integrations: [Sentry.captureConsoleIntegration({ handled: false })],
});
```

`requestDataIntegration` no longer copies `request.user` into Express events.
Call `Sentry.setUser()` explicitly, such as from authentication middleware.

## Feedback migration (9.0.0-guide)

Replace removed `captureUserFeedback()` calls with `captureFeedback()`. Rename
the payload's `comments` property to `message`:

```js
Sentry.captureFeedback({ message: "What happened" });
```

## Additional captured failures (9.0.0)

- Use the browser Statsig integration when Statsig context belongs in browser
  telemetry.
- The Node SDK captures exceptions from `worker_threads`; account for those
  events in filtering and alerting.

## Log and Replay option migration (10.0.0-guide)

Move log options out of `_experiments`:

```js
Sentry.init({
  enableLogs: true,
  beforeSendLog: (log) => log,
});
```

Remove Replay's `_experiments.autoFlushOnFeedback`; user feedback flushes an
active replay by default.

## Structured logger (structured-logs)

`Sentry.logger` emits required-message logs at `trace`, `debug`, `info`, `warn`,
`error`, and `fatal` levels. Supply searchable attributes as the second
argument:

```js
Sentry.logger.info("Order created", { orderId: "order_456" });
```

Use the `Sentry.logger.fmt` tagged template to turn interpolated values into
searchable attributes:

```js
Sentry.logger.info(
  Sentry.logger.fmt`User ${userId} purchased ${productName}`,
);
```

## Shared log and metric attributes

Since 10.61.0, `Sentry.setAttribute()` and `Sentry.setAttributes()` attach
shared attributes to all logs and metrics. Values may be strings, numbers,
booleans, or arrays of those primitive types.

The same methods on the global scope set app-wide attributes. Methods on the
current scope set operation-local attributes:

```js
Sentry.setAttributes({ org_id: user.orgId, user_tier: user.tier });

Sentry.withScope((scope) => {
  scope.setAttribute("request_id", req.id);
  Sentry.logger.info("Processing order");
});
```

## Console and Consola ingestion

`consoleLoggingIntegration({ levels })` converts only the selected console
methods to Sentry logs:

```js
Sentry.init({
  integrations: [
    Sentry.consoleLoggingIntegration({ levels: ["log", "warn", "error"] }),
  ],
});
```

Since 10.13.0, extra console arguments become searchable
`message.parameter.N` attributes. Since 10.12.0, Consola applications can
attach `Sentry.createConsolaReporter()` instead.

## Trace and replay correlation

Logs emitted during an active span include `sentry.trace.parent_span_id`,
supporting trace navigation and filtering. In supported browser environments,
logs emitted during an active Session Replay also include `sentry.replay_id`.

## Framework data controls (modern-server-frameworks)

Elysia, Hono, Nitro, and TanStack Start SDKs accept `dataCollection` options
for automatic request enrichment. Explicitly disable default user information
and all HTTP request bodies when they must not leave the application:

```js
Sentry.init({
  dataCollection: { userInfo: false, httpBodies: [] },
});
```
