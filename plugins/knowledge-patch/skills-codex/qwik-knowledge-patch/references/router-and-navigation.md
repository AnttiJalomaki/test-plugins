# Router and navigation

## Router setup

Qwik City is Qwik Router in V2 and uses `@qwik.dev/router`. `useQwikRouter()`
replaces the early `QwikRouterProvider`-only setup by providing immediate router
context without a provider component. There is one important root-component
qualification: the hook runs only once during SSR. A root that reads reactive
signals should still render `<QwikRouterProvider>`.

The `qwikRouter` middleware owns its router configuration and no longer needs a
separate `qwikRouterConfig` value.

Use `createRenderer()` in SSR entry files when a typed Qwik Router wrapper over
core's `renderToStream()` is needed.

## Document head

`DocumentHeadTags` renders tags managed by the router. `head.styles` and
`head.scripts` use shapes aligned with `head.meta` and `head.links`.

Server render functions can include `documentHead` in `serverData`, making it
available to `useDocumentHead()`. The router's `documentHead` also carries the
client manifest hash for cache busting or ETag construction.

When nested routes merge document-head exports:

- an inner plain-object export overrides an outer object;
- function exports continue to execute inner-first.

## SPA view transitions and links

Qwik Router supports View Transitions during SPA navigation, but current V2
behavior makes them opt-in:

```tsx
<QwikRouterProvider viewTransition={true}>
  <RouterOutlet />
</QwikRouterProvider>
```

The framework emits `qviewTransition` when a transition starts.

`Link` renames `prefetchBundle` to `prefetchBundles`. Link data-prefetch
strategies are configurable; choose them according to route-data freshness and
network cost.

On the first render, the router's previous URL is `undefined`. Code that
compares the current and previous location must handle that missing value.

## Preventing navigation

The stable `usePreventNavigate$` API asynchronously blocks SPA navigation. It
falls back to browser dialogs for other unsaved-state navigation. The earlier
`preventNavigate` Vite experiment flag is no longer required.

## Rewrites and redirects

`RequestEvent.rewrite()` performs an internal redirect while keeping the
browser-visible URL. Returning or throwing its result from a request handler
has the same control-flow effect in current V2 code:

```ts
export const onRequest: RequestHandler = async ({ rewrite }) => {
  return rewrite('/articles/42');
};
```

The stable `request.rewrite()` form also no longer needs an experimental flag.
Multiple rewrite routes may target the same destination route.

An invalid absolute URL passed to rewrite handling returns `400`, not `500`.
Update error handling and tests to expect a client-error response.

`rewriteRoutes` accepts `exclude` patterns for paths that must not receive
generated localized routes.

## Error pages and layouts

Use `404.tsx` for not-found responses. It renders during development as well as
production. Use `error.tsx` for other HTTP statuses and read the current code
with `useHttpStatus()`.

Both error route types render inside layouts selected by normal `@layout` and
`!` modifiers. A miss uses the nearest `404.tsx`. Static builds prerender the
not-found page; rename it to `404!.tsx` when it must bypass the surrounding
layout.

The `notFound` values returned by router factory functions are inert. Define
not-found behavior through the router exports that own it.

## Router development behavior

Router development uses the same Node middleware as production and discovers
new routes through hot reload. Verify navigation in development as well as a
production or SSG build, especially for `404.tsx`, localized rewrites, and
layout modifiers.
