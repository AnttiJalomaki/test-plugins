# Loaders, Server Rendering, Hydration, and Runtime Assets

## Background and blocking stale reloads

A loader may use the object form with `handler` and `staleReloadMode`.
Successful matches whose data is stale reload in `'background'` mode by
default; existing `loaderData` stays visible while the new value is fetched.

Choose `'blocking'` on one loader, or set `defaultStaleReloadMode` on the
router, when navigation must wait:

```tsx
export const Route = createFileRoute('/posts')({
  loader: {
    handler: () => fetchPosts(),
    staleReloadMode: 'blocking',
  },
})
```

`staleTime: Infinity` solves a different problem: it prevents data from
becoming stale rather than selecting the execution mode for stale data.

## Cache defaults and opt-outs

Loader caching uses these defaults:

- Results loaded by navigation have `staleTime: 0`.
- Preloads remain fresh for 30 seconds.
- Unused entries are garbage-collected after 30 minutes.
- `router.invalidate()` reloads active routes immediately and marks all cached
  routes stale.

To discard data when a route unloads while still allowing entry and dependency
loads, combine `gcTime: 0` with `shouldReload: false`. The separate default
`preloadGcTime` still permits a preload to live until its navigation.

```tsx
export const Route = createFileRoute('/posts')({
  loaderDeps: ({ search }) => ({ page: search.page }),
  loader: ({ deps }) => fetchPosts(deps),
  gcTime: 0,
  shouldReload: false,
})
```

When an external cache should receive and deduplicate every loader event, set
`defaultPreloadStaleTime: 0`:

```tsx
const router = createRouter({
  routeTree,
  defaultPreloadStaleTime: 0,
})
```

## Router-native server rendering

The standalone React server-rendering API is experimental. Export a shared
router factory. Pass the web-standard request and factory to
`createRequestHandler`, then hydrate through `RouterClient`.

```tsx
// entry-server.tsx
import {
  createRequestHandler,
  defaultRenderHandler,
} from '@tanstack/react-router/ssr/server'
import { createRouter } from './router'

export function render({ request }: { request: Request }) {
  return createRequestHandler({ request, createRouter })(defaultRenderHandler)
}
```

```tsx
// entry-client.tsx
import { RouterClient } from '@tanstack/react-router/ssr/client'
import { hydrateRoot } from 'react-dom/client'

const router = createRouter()
hydrateRoot(document, <RouterClient router={router} />)
```

The default rendering path creates server memory history and automatically
dehydrates and rehydrates resolved loader data.

The handler returns a web-standard `Response`. Express and other adapters must
translate their request and response objects at the integration boundary.

If custom wrappers or providers must be rendered explicitly, use
`renderRouterToString` with an explicit `RouterServer` instead of
`defaultRenderHandler`.

## Streaming server rendering

Streaming uses the same request-handler shape with `defaultStreamHandler`,
which sends markup and dehydration data automatically:

```tsx
import {
  createRequestHandler,
  defaultStreamHandler,
} from '@tanstack/react-router/ssr/server'

export function render({ request }: { request: Request }) {
  return createRequestHandler({ request, createRouter })(defaultStreamHandler)
}
```

The lower-level alternative is `renderRouterToStream` with an explicit
`RouterServer` child.

## Serialization limits

The built-in serializer handles ordinary JSON data plus:

- `undefined`
- `Date`
- `Error`
- `FormData`

It does not include default round-tripping for `Map`, `Set`, `BigInt`, or other
complex values. Handle those values explicitly before dehydration. A general
serializer customization mechanism remains work in progress.

## Hydration ordering

Custom router hydration runs before the first client route match. This allows
hydrated, request-specific configuration such as URL rewrites to be installed
before server-rendered and client matches are compared.

## Deferred hydration boundaries

TanStack Start's compiler can split `Hydrate` boundaries into client chunks,
preload those generated chunks, preserve their server-rendered fallback HTML,
and replay interaction-triggered events after hydration. This integration
works with both Vite and Rsbuild.

## Runtime server-rendered assets

TanStack Start server rendering supports inline CSS manifests that hydrate
without adding duplicate stylesheet links. `transformAssets` supports both
runtime-configurable inline CSS and opt-in CSS URL templates.

For Rsbuild output, client scripts are modules by default, with IIFE output
available for classic-script environments. A `transformAssets` script callback
receives only:

```ts
{ kind: 'script', url }
```

Configure script asset cross-origin behavior under the `script` key.

## React Server Components

The `@tanstack/react-router` package root defines a `react-server` condition
that keeps the regular API surface while resolving `notFound` and `redirect`
from a server-safe entry. Server Components and server functions can continue
to import them from the package root:

```tsx
import { notFound, redirect } from '@tanstack/react-router'
```
