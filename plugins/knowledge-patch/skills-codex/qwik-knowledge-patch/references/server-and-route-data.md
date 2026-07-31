# Server and Route Data

Use this reference for server-function failures, request-event behavior,
runtime origins, redirect responses, and internal rewrites.

## Server-function error flow

*Batch: `v1.8-1.13`*

Errors are standardized across `server$` functions and route loaders.
`server$` failures can be caught by `@plugin` middleware, so central
middleware can observe or translate these failures.

For client calls:

- 4xx statuses throw;
- statuses above 500 throw; and
- 499 is accepted as a valid status.

Preserve these exact boundaries in tests. Do not write an assertion such as
“all statuses below 500 return normally,” because 4xx failures throw and 499
has explicit accepted-status handling.

Test the same failure at three layers:

1. the server function or route loader creates the intended status;
2. `@plugin` middleware catches the expected failure; and
3. the client call observes a thrown error where required.

Component error boundaries address rendering failures and are documented in
[Components and events](components-and-events.md#error-boundaries).

## Redirect response middleware

*Batch: `v1.8-1.13`*

The send-request event receives a `Response` object even when the request
redirects. Middleware can inspect the redirect response through the same
response-shaped interface as a non-redirecting request.

Review middleware that treated redirects as an absent response or a separate
sentinel. It should now branch on response status and headers.

## Bun and Deno request origins

*Batch: `v1.14-1.19`*

`QwikCityBunOptions` and `QwikCityDenoOptions` accept `getOrigin` to control
URL-origin handling.

Use it behind proxies or nonstandard runtimes where the origin cannot be
derived safely from the raw request. The callback should return the
application's externally meaningful origin, including the correct scheme and
host.

## Request-event immutability

*Batch: `v1.14-1.19`*

Request events use readonly types instead of being frozen at runtime.
TypeScript prevents supported code from mutating readonly properties, but
runtime code must not use `Object.isFrozen()` or mutation failure as a feature
test.

Treat the event as immutable through the public API. If middleware needs
derived state, store it separately or use the framework's supported request
interfaces.

## Request-event rewrites

*Batch: `v1.14-1.19`*

`RequestEvent.rewrite()` internally redirects processing while preserving the
browser-visible URL. Throw the returned value:

```ts
export const onRequest: RequestHandler = async ({ rewrite }) => {
  throw rewrite('/articles/42');
};
```

Do not replace this with a client-visible redirect when the original URL must
remain in the address bar. Multiple source rewrites may share one destination.

The routing implications are documented in
[Router and navigation](router-and-navigation.md#request-event-rewrites).

## Request and redirect verification

When changing request middleware:

- cover direct, redirected, and rewritten requests;
- assert the `Response` seen by the send-request event;
- verify the browser-visible URL after a rewrite;
- exercise the runtime-specific `getOrigin` callback;
- avoid runtime-freeze assumptions for readonly request events; and
- confirm redirect responses default to `no-store` rather than inheriting a
  parent layout cache header.
