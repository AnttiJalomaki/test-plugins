---
name: tanstack-router-knowledge-patch
description: TanStack Router
version: 1.170.18
license: MIT
metadata:
  author: Nevaberry
---


# TanStack Router Knowledge Patch

Use this skill when implementing, reviewing, or debugging TanStack Router and
TanStack Start applications. Start with the quick references below, then open
the topic file that matches the work.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/search-and-matching.md](references/search-and-matching.md) | Search validation and typing, search middleware, route matching, parameter parsing, and route-level errors |
| [references/rewrites-masks-and-blocking.md](references/rewrites-masks-and-blocking.md) | Bidirectional URL rewrites, basepaths, route masks, reload behavior, and navigation blockers |
| [references/loaders-and-ssr.md](references/loaders-and-ssr.md) | Loader freshness and cache behavior, server rendering, streaming, serialization, hydration, and runtime assets |
| [references/code-splitting-and-tooling.md](references/code-splitting-and-tooling.md) | File and code route splitting, virtual routes, plugin configuration, Rsbuild, transforms, HMR, and intent tooling |

## Working method

1. Inspect the installed Router, Start, adapter, and bundler-plugin versions.
2. Identify whether the application uses file routes, code routes, or both.
3. Keep match-critical route configuration in the eager route module.
4. Treat the address-bar URL and the internal matched URL as separate values
   when rewrites or masks are enabled.
5. Decide explicitly whether stale loader refreshes should block navigation.
6. Test direct loads, client navigation, reloads, and server rendering; each can
   exercise a different URL or hydration path.
7. Prefer the project's generated route tree, types, tests, and observed runtime
   behavior when they disagree with assumptions.

## Compatibility traps

### Do not rely on Zod v3 `.catch()` for navigation input types

Reads use a search validator's output type, but links and navigation use its
input type. A default only makes a field optional at navigation sites when both
sides are preserved. For Zod v3, use `@tanstack/zod-adapter` and `fallback`;
passing Zod v3 `.catch()` directly can erase the input/output distinction.

```tsx
import { fallback, zodValidator } from '@tanstack/zod-adapter'

const searchSchema = z.object({
  page: fallback(z.number(), 1).default(1),
})

export const Route = createFileRoute('/products')({
  validateSearch: zodValidator(searchSchema),
})

const link = <Link to="/products" />
```

Zod v4 and Standard Schema implementations can be supplied directly. A thrown
validation failure reaches `onError` with `routerCode` set to
`'VALIDATE_SEARCH'` and renders the route's `errorComponent`.

### Do not expect the CLI alone to perform automatic splitting

`autoCodeSplitting` belongs to a bundler plugin. Put the router plugin before
the framework plugin:

```ts
plugins: [
  tanstackRouter({ autoCodeSplitting: true }),
  react(),
]
```

Automatic splitting extracts only `component`, `errorComponent`,
`pendingComponent`, and `notFoundComponent`. Loaders, `beforeLoad`, validation,
context, static data, links, scripts, styles, and matching configuration stay
in the critical chunk.

For manual file-route splitting, keep critical options in the ordinary route
file and place only render options in the matching `.lazy.tsx` file:

```tsx
// routes/posts.tsx
export const Route = createFileRoute('/posts')({ loader: fetchPosts })

// routes/posts.lazy.tsx
export const Route = createLazyFileRoute('/posts')({ component: Posts })
```

The `__root` route cannot be split. If a file route has no eager configuration,
remove its empty normal file and let the generated tree provide its virtual
anchor.

### Distinguish stale reload policy from freshness

The object loader form accepts `handler` and `staleReloadMode`. Successful stale
matches reload in the background by default, leaving current `loaderData`
visible. Select `'blocking'` on a loader, or set `defaultStaleReloadMode`, when
navigation must await the replacement:

```tsx
export const Route = createFileRoute('/posts')({
  loader: {
    handler: () => fetchPosts(),
    staleReloadMode: 'blocking',
  },
})
```

`staleTime: Infinity` prevents staleness; it does not change the behavior of a
reload that is already stale.

### Preserve both sides of a URL rewrite

Router `rewrite.input` converts a public browser URL to the internal URL used
for matching. `rewrite.output` converts internal destinations back before a
link or history entry is written. Consequently, `location.href` is internal
and `location.publicHref` is shareable.

```tsx
const localeRewrite = {
  input: ({ url }) => {
    url.pathname =
      url.pathname.replace(/^\/(en|fr)(?=\/|$)/, '') || '/'
    return url
  },
  output: ({ url }) => {
    url.pathname = `/en${url.pathname === '/' ? '' : url.pathname}`
    return url
  },
}

const router = createRouter({ routeTree, rewrite: localeRewrite })
```

Handlers may return the mutated `URL`, another `URL`, an href string, or
`undefined`. Client links, programmatic navigation, server request parsing, and
hydration use the same rewrite configuration. An output rewrite that changes
the origin turns a `<Link>` into a hard navigation.

### Treat a mask as history state, not a redirect

A route mask matches and renders one typed runtime location while displaying
another URL. The hidden location lives in browser history state. Copying the
displayed URL loses it and loads the displayed route normally; a local reload
retains it unless `unmaskOnReload` says otherwise.

```tsx
const photoMask = createRouteMask({
  routeTree,
  from: '/photos/$photoId/modal',
  to: '/photos/$photoId',
  params: (prev) => ({ photoId: prev.photoId }),
})

const router = createRouter({
  routeTree,
  routeMasks: [photoMask],
  unmaskOnReload: true,
})
```

Per-link or per-navigation `unmaskOnReload` wins over a route-mask value, which
wins over the router default.

## High-value features

### Compose search transformations through `next`

`search.middlewares` run for links to a route or its descendants and run again
after validation during navigation. `retainSearchParams` carries selected
current values; `stripSearchParams` removes values equal to defaults.

```tsx
search: {
  middlewares: [
    retainSearchParams(['campaign']),
    stripSearchParams({ page: 1, tags: [] }),
  ],
}
```

### Compose rewrites in inverse output order

`composeRewrites` runs inputs first-to-last and outputs last-to-first. A
configured `basepath` is outside custom rewrites: Router strips it before
custom input and restores it after custom output.

### Choose blocker semantics deliberately

With `withResolver: true`, a true `shouldBlockFn` result enters a blocked state;
the UI must call `proceed` or `reset`. `enableBeforeUnload` separately controls
the native reload and tab-close prompt.

Without resolver mode, `shouldBlockFn` may return a promise. Resolve `true` to
cancel navigation and `false` to allow it.

### Use the web-standard SSR boundary

Export a shared router factory. On the server, pass it and a web `Request` to
`createRequestHandler`; hydrate the client with `RouterClient`. The default
render path supplies memory history and transfers resolved loader data.

```tsx
export function render({ request }: { request: Request }) {
  return createRequestHandler({ request, createRouter })(defaultRenderHandler)
}
```

The handler returns a web `Response`, so non-web adapters must translate at
their boundary. Use `defaultStreamHandler` for automatic markup and dehydration
streaming. Use `renderRouterToString` or `renderRouterToStream` with an explicit
`RouterServer` when custom wrappers or providers are required.

### Know the loader cache defaults

- Navigation results have `staleTime: 0`.
- Preloads remain fresh for 30 seconds.
- Unused entries are collected after 30 minutes.
- `router.invalidate()` immediately reloads active routes and marks every
  cached route stale.
- `gcTime: 0` with `shouldReload: false` discards unloaded data while still
  permitting entry and dependency loads.
- `defaultPreloadStaleTime: 0` lets an external cache observe and deduplicate
  every loader event.

## Review checklist

- Validate malformed URLs with tolerant fallbacks if navigation should
  continue.
- Confirm validator input and output types at both `<Link>` and route-read
  sites.
- Check middleware order and whether it affects descendants.
- Test ambiguous static, dynamic, optional, wildcard, and prioritized routes.
- Verify both rewrite directions and direct server requests under a basepath.
- Test masked navigation, copied URLs, and reload behavior separately.
- Keep only supported render options behind file-route lazy boundaries.
- Check loader freshness, garbage collection, invalidation, and external-cache
  interaction independently.
- Keep SSR payloads within the built-in serializer's supported value types.
- Confirm build plugin order, virtual-route punctuation, aliases, and HMR with
  the actual bundler.
