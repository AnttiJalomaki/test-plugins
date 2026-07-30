# Data loading and server rendering

## Background and blocking stale reloads

A loader can use object form with `handler` and `staleReloadMode`:

```tsx
export const Route = createFileRoute('/posts')({
  loader: {
    handler: () => fetchPosts(),
    staleReloadMode: 'blocking',
  },
})
```

When a successful match is stale, the default mode is `'background'`. Existing
`loaderData` stays visible while revalidation runs. Use `'blocking'` when the
navigation must wait for the replacement result. Configure it per loader or set
`defaultStaleReloadMode` on the router.

Do not use `staleTime: Infinity` as a substitute for reload mode. It prevents
data from becoming stale instead of controlling what happens once stale data is
reloaded.

## Cache defaults and opt-outs

Router loader caching uses these defaults:

- navigation results have `staleTime: 0`;
- preloads remain fresh for 30 seconds;
- unused entries are garbage-collected after 30 minutes;
- `router.invalidate()` reloads active routes immediately and marks every cached
  route stale.

To discard loader data after a route unloads but still permit entry and
dependency loads, combine `gcTime: 0` with `shouldReload: false`:

```tsx
export const Route = createFileRoute('/posts')({
  loaderDeps: ({ search }) => ({ page: search.page }),
  loader: ({ deps }) => fetchPosts(deps),
  gcTime: 0,
  shouldReload: false,
})
```

The separate default `preloadGcTime` still lets preloaded data survive until a
navigation consumes it. When an external cache should see and deduplicate every
loader event, set `defaultPreloadStaleTime: 0`:

```tsx
const router = createRouter({
  routeTree,
  defaultPreloadStaleTime: 0,
})
```

## Router-native React SSR

The standalone React SSR API is experimental. Export a shared router factory,
then give that factory and a web-standard `Request` to `createRequestHandler`.
The default path supplies server memory history and dehydrates resolved loader
data for `RouterClient` to rehydrate.

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

The request handler returns a web-standard `Response`. Express and similar
adapters must translate framework request and response objects at the boundary.

When the default render path is too restrictive, call `renderRouterToString`
with an explicit `RouterServer` child so custom wrappers or providers can be
rendered.

## Streaming SSR

Streaming uses the same request-handler pattern with `defaultStreamHandler`,
which streams both markup and dehydration data:

```tsx
import {
  createRequestHandler,
  defaultStreamHandler,
} from '@tanstack/react-router/ssr/server'

export function render({ request }: { request: Request }) {
  return createRequestHandler({ request, createRouter })(defaultStreamHandler)
}
```

For lower-level control, use `renderRouterToStream` with an explicit
`RouterServer` child.

## Serialization boundaries

The built-in SSR serializer round-trips:

- ordinary JSON-compatible data;
- `undefined`;
- `Date`;
- `Error`;
- `FormData`.

`Map`, `Set`, `BigInt`, and other complex values are not supported by default.
Convert them explicitly before dehydration and restore them after hydration.
A general serializer-customization mechanism remains work in progress.
