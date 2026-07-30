# Telemetry Data, Sessions, and Logs

## Session tracking

The v9 migration removes `autoSessionTracking`. Session sources are now
integration-specific:

- browser sessions use `browserSessionIntegration`;
- server request sessions use `httpIntegration`;
- Node.js process sessions use the default `processSessionIntegration`.

Disable browser tracking by removing its integration. Disable server request
tracking with:

```js
Sentry.init({
  integrations: [
    Sentry.httpIntegration({ trackIncomingRequestsAsSessions: false }),
  ],
});
```

Since `9.0.0`, core always selects the session on the isolation scope. This can
change the selected session when more than one scope contains session state.

## Browser and request data

Browser SDKs no longer ask the backend to infer an IP address by default. Use
`sendDefaultPii: true` only when IP inference is intended.

With `attachStackTrace: true`, `captureConsoleIntegration` marks console events
handled by default. Pass `{ handled: false }` when they should remain unhandled:

```js
Sentry.init({
  sendDefaultPii: true,
  attachStackTrace: true,
  integrations: [Sentry.captureConsoleIntegration({ handled: false })],
});
```

`requestDataIntegration` no longer copies Express `request.user` into events.
Call `Sentry.setUser()` explicitly from application middleware.

The Elysia, Hono, Nitro, and TanStack Start SDKs expose a common explicit
opt-out for automatic user and request-body enrichment:

```js
Sentry.init({
  dataCollection: { userInfo: false, httpBodies: [] },
});
```

## Stack-frame variables

As of `10.68.0`, `stackFrameVariables` supports filtering by variable name.
Use it to keep only the captured variables permitted by the application's data
policy.

## Structured logger

The `structured-logs` API provides required-message methods at `trace`,
`debug`, `info`, `warn`, `error`, and `fatal`. Supply searchable attributes as
the second argument:

```js
Sentry.logger.info("Order created", { orderId: "order_456" });
```

`Sentry.logger.fmt` is a tagged template that extracts interpolated values as
searchable attributes:

```js
Sentry.logger.info(
  Sentry.logger.fmt`User ${userId} purchased ${productName}`,
);
```

In v10, logs configuration is no longer experimental. Move
`_experiments.enableLogs` and `_experiments.beforeSendLog` to top-level
initialization options:

```js
Sentry.init({
  enableLogs: true,
  beforeSendLog: log => log,
});
```

Replay's `_experiments.autoFlushOnFeedback` is removed because feedback now
flushes Replay automatically.

## Shared log and metric attributes

Since 10.61.0, `Sentry.setAttribute()` and `Sentry.setAttributes()` attach
shared attributes to logs and metrics. Values may be strings, numbers,
booleans, or arrays of those primitives. The same methods on the global or
current scope provide app-wide or operation-local attributes.

```js
Sentry.setAttributes({ org_id: user.orgId, user_tier: user.tier });
Sentry.withScope(scope => {
  scope.setAttribute("request_id", req.id);
  Sentry.logger.info("Processing order");
});
```

## Console and Consola ingestion

`consoleLoggingIntegration({ levels })` converts selected console calls into
logs. Since 10.13.0, extra console arguments become searchable
`message.parameter.N` attributes.

```js
Sentry.init({
  integrations: [
    Sentry.consoleLoggingIntegration({ levels: ["log", "warn", "error"] }),
  ],
});
```

Consola users can attach `Sentry.createConsolaReporter()` since 10.12.0.

## Trace and Replay correlation

Logs emitted during an active span automatically carry
`sentry.trace.parent_span_id`. In supported browsers, logs emitted during an
active Session Replay also carry `sentry.replay_id`.
