# Router and Navigation

Use this reference for SPA prevention, previous-location handling, rewrite
topology, internal redirects, and redirect caching.

## MPA-only and navigation-prevention experiments

*Batch: `v1.8-1.13`*

Enable routing experiments through `qwikVite()`:

```ts
qwikVite({
  experimental: ['noSPA', 'preventNavigate'],
});
```

Use `noSPA` only for an MPA-only application that does not use `Link`. It is
not a generic optimization flag for applications that retain SPA navigation.

`preventNavigate` enables `usePreventNavigate`. The hook can asynchronously
block SPA navigation. For other navigation with unsaved state, it falls back
to the browser's dialog behavior.

Test navigation initiated by links, browser history, and full document exits.
An asynchronous SPA decision and a browser-controlled unload prompt do not
have identical UX or test behavior.

The same experimental array also supports `valibot` for the experimental
`valibot$` validator; see
[Build and deployment](build-and-deployment.md#experimental-feature-gating).

## Initial previous URL

*Batch: `v1.8-1.13`*

On the first render, the router's previous URL is `undefined`. Treat absence
as the initial-navigation state.

Do not substitute the current URL unless the application explicitly wants that
semantic. Components that compare previous and current locations should have a
separate first-render branch.

## Rewrite fan-in

*Batch: `v1.8-1.13`*

Multiple rewrite routes may point to the same destination route. This is valid
fan-in and no longer produces an error.

Use it for aliases, localization entry points, or legacy paths that share one
handler. Preserve the original browser-visible URL when application behavior,
analytics, or canonical links depend on it.

## Request-event rewrites

*Batch: `v1.14-1.19`*

`RequestEvent.rewrite()` performs an internal redirect while preserving the
URL shown in the browser. Throw its result from a request handler:

```ts
export const onRequest: RequestHandler = async ({ rewrite }) => {
  throw rewrite('/articles/42');
};
```

Because the visible URL remains unchanged, generate canonical metadata and
relative links with an explicit understanding of whether they should describe
the requested path or the rewritten destination.

Request-event typing and middleware details are in
[Server and route data](server-and-route-data.md#request-event-rewrites).

## Redirect caching

*Batch: `v1.14-1.19`*

Redirect responses no longer inherit `Cache-Control` from a parent layout.
They default to `no-store`.

Set a redirect cache policy explicitly only when the redirect is safe to cache
for every affected user and request variant. Do not expect a layout's page
cache policy to carry over.

Route-data requests have a different cache path: Qwik City navigation follows
the `q-data.json` response headers and uses a one-hour default duration. See
[Build and deployment](build-and-deployment.md#asset-paths-and-cache-headers).

## Related transition lifecycle

Qwik emits `qviewTransition` when a view transition starts. See
[Components and events](components-and-events.md#view-transition-event) when
navigation code needs to coordinate with that event.
