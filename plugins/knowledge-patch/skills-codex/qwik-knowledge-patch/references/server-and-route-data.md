# Server functions, requests, and route data

## Server-function errors and control flow

Errors from `server$` functions and route loaders use a standardized flow. A
`server$` failure can be caught by `@plugin` middleware. Client calls throw for
4xx statuses and statuses above 500, and status 499 is accepted as valid.

When a non-`ServerError` escapes `server$()`, it is logged on the server.

Returning any of these values from a loader, action, request handler, or server
function has the same control-flow effect as throwing it:

```ts
return ev.redirect(302, '/login');
return ev.error(403, 'Forbidden');
return ev.rewrite('/internal/path');
```

This supports direct returns without changing redirect, error, or rewrite
semantics.

## Request events and middleware

Request-event types are readonly, but their objects are no longer frozen at
runtime. Treat readonly typing as the API contract rather than depending on
runtime `Object.freeze` behavior.

The send-request event receives a `Response` object even when the upstream
request redirects.

Middleware can inspect `RequestEvent.internalRequest` to distinguish
framework-internal JSON requests. Server request-body limits are configurable;
set them according to endpoint needs instead of assuming one global payload
size is suitable.

`RequestEvent.rewrite()` preserves the visible URL while routing internally.
Invalid absolute rewrite URLs produce `400`. Redirects do not inherit a parent
layout's `Cache-Control` header and default to `no-store`.

## Request origins and locale

`QwikCityBunOptions` and `QwikCityDenoOptions` accept `getOrigin` to control
the origin used when constructing request URLs behind proxies or adapters.

When the runtime provides it, server-side `withLocale()` uses
`AsyncLocalStorage`. The request locale therefore remains available throughout
asynchronous work spawned in that locale context.

## Route-loader signals and failures

The expanded `routeLoader$` signal type and its ESLint rule recognize broader
signal usage in V2.

Route loaders follow async-signal error semantics:

- thrown failures populate `.error`;
- reading `.value` after such a failure rethrows;
- calling `fail()` is distinct and sets `.value` to `{ failed }`;
- `expires`, `poll`, and `allowStale` control result freshness.

A `routeLoader$` cannot read action state. Read the action signal in the
component, or encode relevant state in the URL so the loader can consume it.

## Query-driven reruns

The loader `search` option declares the query parameters it receives and that
trigger reruns. `qwikRouter` defaults `strictLoaders` to `true`, so a loader
without `search` receives no query parameters and does not rerun merely because
the query string changed.

Declare the smallest input surface explicitly:

```ts
export const useResults = routeLoader$(({ query }) => {
  return loadResults(query.get('q'));
}, { search: ['q'] });
```

## Loader transport and browser caching

SPA navigation uses manifest-versioned
`q-loader-${hash}.${manifestHash}.json` data instead of requesting
`q-data.json` on every navigation.

Loader `expiry` defaults to two minutes. Setting `expiry: 0` disables browser
caching and, during SSG, emits the loader data as a static file. Expiring
responses are private by default. User-specific data should still use a short
expiry and ETag validation.

## Loader ETags and cache keys

Route loaders accept `eTag` and `cacheKey`. The current cache-key callback
signature is:

```ts
cacheKey(requestEv: RequestEvent, eTag: string)
```

Do not use the earlier `(status, eTag, pathname)` signature.

For SSR, when `routeConfig` supplies `cacheKey` but no `eTag`, Qwik hashes the
rendered output to create the ETag.

## Background loaders

`blockSSR` defaults to `true`. With the experimental `blockSSR` feature enabled,
set a loader's `blockSSR` to `false` to run it in the background without
delaying server rendering. Ensure the component handles pending and failure
states because the HTML can arrive first.

## Page ETags and in-memory HTML caching

A page module can export `eTag` to emit an ETag header and let the prerender
check return `304`. It can export `cacheKey` to enable an in-memory cache of
rendered HTML. Request-handler middleware can invalidate that cache with
`clearSsrCache`.

Group page behavior under `routeConfig` when appropriate:

```ts
export const routeConfig = {
  head,
  eTag,
  cacheKey,
};
```

`routeConfig` may be a static object or function and resolves with the same
rules as `head`. When present, it takes precedence over separate `head`, `eTag`,
and `cacheKey` exports from that module.

## SSG cache and route-plan implications

Router SSG can rerun against existing server output through
`server/run-ssg.js`. In the Vite app-builder flow, fully prerendered routes that
require no server are left out of the production SSR route plan.

Test cache behavior in both SSR and SSG output: a static loader file from
`expiry: 0`, a page-level ETag, and a route-loader transport ETag serve distinct
purposes and should not share user-specific content accidentally.
