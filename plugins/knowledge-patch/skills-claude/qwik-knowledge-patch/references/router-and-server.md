# Router and Server Behavior

Use this reference for Qwik Router context, navigation, middleware, route
loaders, document head, errors, rewrites, caching, and rendering.

Relevant source batches are `v1.8-1.13`, `v1.14-1.19`,
`v2-alpha-beta-1-9`, `v2-beta-10-19`, `v2-beta-20-29`,
`v2-beta-30-38`, and `migration-v1-v2`.

## Router context and rendering

Qwik City is named Qwik Router in V2. `useQwikRouter()` replaced the earlier
provider-only access pattern and makes Router context immediately available.

When removing V1's `<QwikCityProvider>`, a non-reactive root can call
`useQwikRouter()` directly while rendering `RouterOutlet`. The hook runs only
once during SSR, so a root component that reads signals must instead use
`<QwikRouterProvider>`.

`createRenderer()` wraps Core's `renderToStream()` with Router types for SSR
entry files.

The `qwikRouter` middleware now handles its own Router configuration; it no
longer needs a separate `qwikRouterConfig`.

## Document head

`DocumentHeadTags` renders Router-managed tags. `head.styles` and
`head.scripts` use shapes aligned with `head.meta` and `head.links`.

Server rendering can pass `documentHead` through `serverData`, making it
available to `useDocumentHead()`. Later `documentHead` data includes the client
manifest hash for cache busting or ETag generation.

When nested routes merge head exports, inner plain-object exports override
outer objects. Function exports continue to execute inner-first.

## Navigation and transitions

Qwik Router supports View Transitions during SPA navigation, but the current
default is opt-in:

```tsx
<QwikRouterProvider viewTransition={true}>
  <RouterOutlet />
</QwikRouterProvider>
```

Qwik emits the `qviewTransition` custom event when a view transition starts
and a `qrender` event after each render.

`Link.prefetchBundle` is renamed `prefetchBundles`. Link data-prefetch
strategies are configurable.

On the first render, the Router previous URL is `undefined`. Consumers must
handle the missing value.

`usePreventNavigate$()` is stable. It asynchronously blocks SPA navigation;
other attempts to leave unsaved state fall back to browser dialogs.

## Redirect, error, and rewrite control flow

Returning `ev.redirect()`, `ev.error()`, or `ev.rewrite()` from a loader,
action, request handler, or server function now has the same effect as throwing
the returned control-flow value.

`RequestEvent.rewrite()` performs an internal redirect while preserving the
browser-visible URL:

```ts
export const onRequest: RequestHandler = async ({ rewrite }) => {
  throw rewrite('/articles/42');
};
```

Multiple rewrite routes may target the same destination. An invalid absolute
URL passed to rewrite handling produces `400`, not `500`.

The send-request event receives a `Response` even when the request redirects.
Redirects do not inherit a parent layout's `Cache-Control` and default to
`no-store`.

## Error routes and layouts

Custom `404.tsx` pages render in development and production. `error.tsx`
handles other HTTP statuses and can read the current code through
`useHttpStatus()`.

`404.tsx` and `error.tsx` resolve layouts according to `@layout` and `!`
modifiers, and a miss selects the nearest `404.tsx`. Static builds prerender
the not-found page. Rename it to `404!.tsx` when it must bypass its layout.

The `notFound` exports from Router factory functions are inert; the `router`
exports own not-found handling.

## Server functions and request context

Server-function and route-loader errors use standardized flow. `server$`
failures can be caught by `@plugin` middleware. Client calls throw for 4xx
statuses and statuses above 500, and `499` is accepted.

A non-`ServerError` thrown by `server$()` is logged on the server.

When available, `withLocale()` uses `AsyncLocalStorage` on the server, so
asynchronous work retains the request locale.

Request events use readonly types rather than runtime freezing.
`RequestEvent.internalRequest` identifies framework-internal JSON requests.
Server request-body limits are configurable.

`QwikCityBunOptions` and `QwikCityDenoOptions` accept `getOrigin` for custom
URL-origin handling.

## Loader result semantics

Route loaders return async-signal-like results:

- a thrown failure populates `.error`;
- reading `.value` rethrows that failure;
- `fail()` is distinct and sets `.value` to `{ failed }`;
- options include `expires`, `poll`, and `allowStale`; and
- a route loader cannot read action state.

Read action state in the component. If it must drive a loader, move the
relevant state into the URL.

The route-loader signal type and its ESLint rule were extended to recognize
broader signal usage.

## Strict query-driven loaders

The loader `search` option declares which query parameters are passed to the
loader and which changes trigger reruns.

`qwikRouter` defaults `strictLoaders` to `true`. A loader without `search`
receives no query parameters and does not rerun for query-string changes.

## Background loaders

`blockSSR` defaults to `true`. With the experimental `blockSSR` feature
enabled, set `blockSSR: false` to run a route loader in the background without
delaying SSR.

## Loader transport caching

SPA navigation fetches manifest-versioned
`q-loader-${hash}.${manifestHash}.json` files rather than requesting
`q-data.json` every time.

Loader `expiry` defaults to two minutes. `expiry: 0` disables browser caching
and emits loader data as a static file during SSG. Expiring responses are
private by default. User-specific data still calls for a short expiry and ETag
validation.

Earlier Qwik City navigation follows the cache headers on `q-data.json`; its
default cache duration is one hour rather than always forcing a fresh
download.

## Page and loader ETags

A page module can export `eTag` to generate an ETag response header and allow a
prerender check to return `304`.

A page `cacheKey` function enables an in-memory cache of rendered HTML.
Request-handler middleware can clear that cache through `clearSsrCache`.

Route loaders also accept `eTag` and `cacheKey`. The loader cache-key signature
is:

```ts
(requestEv: RequestEvent, eTag: string) => string
```

It replaces the older `(status, eTag, pathname)` signature.

## Unified `routeConfig`

A page module may export `routeConfig` as a static object or function. It uses
the same resolution rules as `head` and groups `head`, `eTag`, and `cacheKey`.
When present, it takes precedence over separate exports of those values in the
same module.

For SSR, a `routeConfig` with `cacheKey` but without `eTag` hashes the rendered
output to create the ETag.

## SSG and localized routing

`rewriteRoutes` accepts `exclude` path patterns that must not receive generated
localized routes.

Fully prerendered, server-free routes are left out of the production SSR route
plan. SSG runs in a dedicated Vite application-build environment; see the
build reference before invoking Vite programmatically.

## Router tests

`QwikCityMockProvider` can mock route loaders and actions. Tests should cover:

- the undefined initial previous URL;
- inner/outer head merge order;
- return and throw control-flow variants;
- invalid absolute rewrites returning `400`;
- strict loader query changes;
- `304` ETag paths and cache clearing;
- custom `404.tsx` and `error.tsx` in development; and
- layout modifiers on error routes.
