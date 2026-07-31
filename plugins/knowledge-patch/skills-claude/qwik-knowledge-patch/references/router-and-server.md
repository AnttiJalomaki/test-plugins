# Router and Server Behavior

## Server-function error flow

By 1.13, `server$` functions and route loaders use standardized error behavior.
A client call throws for 4xx statuses and statuses above 500, while 499 is a
valid status. `@plugin` middleware can catch failures from `server$`.

## Redirect responses in middleware

The send-request event receives a `Response` object even when the request
redirects. Middleware can therefore handle the redirect response through the
same event value.

## Initial previous URL

The router's previous URL is `undefined` on the first render. Treat it as
optional before reading or comparing it.

## Rewrite fan-in

Multiple rewrite routes can point to the same destination route without
causing a route conflict.

## Route-loader and action mocks

`QwikCityMockProvider` can mock route loaders and actions. Use those mocks when
a component test depends on route data or action results.

## Bun and Deno origins

`QwikCityBunOptions` and `QwikCityDenoOptions` accept `getOrigin`. Supply it
when an adapter must control how the request URL origin is determined.

## Request-event immutability

Request events use readonly types rather than runtime freezing. Respect the
readonly contract, but do not depend on the object being frozen at runtime.

## Internal request rewrites

`RequestEvent.rewrite()` performs an internal redirect without changing the URL
shown in the browser. Throw the returned result from the request handler:

```ts
export const onRequest: RequestHandler = async ({ rewrite }) => {
  throw rewrite('/articles/42');
};
```

## Redirect caching

A redirect does not inherit `Cache-Control` from its parent layout. Its default
is `no-store`; set a different policy explicitly only when the redirect should
be cached.

## Route-data caching

Qwik City no longer forces `q-data.json` to download fresh on every navigation.
Navigation obeys its cache headers, whose default duration is one hour.
