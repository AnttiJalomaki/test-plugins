---
name: tanstack-router-knowledge-patch
description: TanStack Router
version: 1.170.18
license: MIT
metadata:
  author: Nevaberry
---


# TanStack Router Knowledge Patch

Load this skill when implementing or reviewing TanStack Router or TanStack Start
code. Begin with the compatibility-critical notes below, then open only the
reference files relevant to the task.

## Reference index

| Reference | Topics |
| --- | --- |
| [search-params.md](references/search-params.md) | Search validation, validator typing, defaults, and search middlewares |
| [rewrites-and-masking.md](references/rewrites-and-masking.md) | Bidirectional URL rewrites, composition, basepaths, and route masks |
| [matching-and-params.md](references/matching-and-params.md) | Segment-trie matching, param priority, parse rejection, and route errors |
| [loading-and-ssr.md](references/loading-and-ssr.md) | Loader freshness, stale reloads, cache lifetime, native SSR, streaming, and serialization |
| [code-splitting-and-navigation.md](references/code-splitting-and-navigation.md) | Automatic and manual route splitting, lazy loaders, route directories, and blockers |
| [start-and-build-tooling.md](references/start-and-build-tooling.md) | Hydration boundaries, SSR assets, RSC exports, virtual routes, plugins, transforms, and HMR |

## Compatibility-critical behavior

### Treat public and internal URLs as different values

Router-level rewrites parse the browser URL with `input` and emit a browser URL
with `output`. A matched location keeps its internal URL in `location.href` and
its shareable external URL in `location.publicHref`.

- Do not display or copy `location.href` when an output rewrite is active.
- Return a mutated `URL`, a new `URL`, a full href string, or `undefined` from a
  rewrite handler.
- Expect links whose output rewrite changes origin to perform a hard navigation.
- Keep the same rewrite setup on the server so request matching and hydration
  agree.

See [rewrites-and-masking.md](references/rewrites-and-masking.md).

### Do not depend on flat-list route ordering

Matching traverses a segment trie. Static branches have priority, ambiguous
dynamic and optional branches are explored deterministically, and wildcards are
considered last. Use `params.priority` only to break a genuine candidate tie.
An experimental `params.parse` may return `false` to reject an incoming
candidate.

See [matching-and-params.md](references/matching-and-params.md).

### Keep matching code out of lazy render chunks

Automatic splitting extracts only these route options:

- `component`
- `errorComponent`
- `pendingComponent`
- `notFoundComponent`

Loaders, `beforeLoad`, search validation, context, static data, links, scripts,
styles, and other matching configuration stay in the critical chunk. Automatic
splitting requires a bundler plugin; the router CLI alone cannot provide it.

See
[code-splitting-and-navigation.md](references/code-splitting-and-navigation.md).

### Distinguish stale reload policy from freshness

Use loader object form when a stale navigation must block:

```tsx
export const Route = createFileRoute('/posts')({
  loader: {
    handler: () => fetchPosts(),
    staleReloadMode: 'blocking',
  },
})
```

The default mode for a stale successful match is `background`, which preserves
existing `loaderData` during revalidation. `staleTime: Infinity` prevents data
from becoming stale; it does not make a stale reload blocking.

See [loading-and-ssr.md](references/loading-and-ssr.md).

### Preserve blocker boolean semantics

`shouldBlockFn` answers whether navigation should be blocked:

- `true` blocks or cancels.
- `false` allows navigation.
- With `withResolver: true`, call `proceed()` to leave or `reset()` to stay.
- Without resolver mode, the callback may return a promise using the same
  boolean meanings.
- `enableBeforeUnload` independently controls the native reload or tab-close
  prompt.

See
[code-splitting-and-navigation.md](references/code-splitting-and-navigation.md).

### Keep unsupported values out of SSR dehydration

The built-in serializer handles ordinary JSON plus `undefined`, `Date`, `Error`,
and `FormData`. It does not provide default round-tripping for `Map`, `Set`,
`BigInt`, or arbitrary complex values. Convert unsupported loader data at an
explicit boundary.

See [loading-and-ssr.md](references/loading-and-ssr.md).

### Preserve literal punctuation in virtual route definitions

Dots in explicit virtual paths and pathless layout IDs are literal. Leading and
trailing underscores in virtual `route()` paths are also literal URL characters.
Physical file routes still require bracket escapes for literal underscore
segments.

See [start-and-build-tooling.md](references/start-and-build-tooling.md).

## Common implementation patterns

### Validate search without making defaulted fields required

Navigation uses a validator's input type; route reads use its output type. Keep
both sides intact when defaults should make search fields optional.

```tsx
import { fallback, zodValidator } from '@tanstack/zod-adapter'

const searchSchema = z.object({
  page: fallback(z.number(), 1).default(1),
})

export const Route = createFileRoute('/products')({
  validateSearch: zodValidator(searchSchema),
})
```

Use the adapter and `fallback` for Zod v3. Zod v4 and Standard Schema
implementations can be passed directly. For malformed URLs that should continue
navigation, use tolerant schema fallbacks; thrown validation errors are routed
through `onError` with `routerCode === 'VALIDATE_SEARCH'`.

See [search-params.md](references/search-params.md).

### Compose search URL policy declaratively

Put search transforms in `search.middlewares`. They affect links to the route or
descendants and run again after validation during navigation. Use
`retainSearchParams` to inherit selected current values and `stripSearchParams`
to omit supplied defaults.

See [search-params.md](references/search-params.md).

### Select a loader cache strategy explicitly

The important defaults are:

- navigation results are immediately stale (`staleTime: 0`);
- preloads remain fresh for 30 seconds;
- unused cache entries are collected after 30 minutes;
- `router.invalidate()` immediately reloads active routes and marks all cached
  routes stale.

Use `gcTime: 0` with `shouldReload: false` to discard data on unload while still
allowing dependency or entry loads. Set `defaultPreloadStaleTime: 0` when an
external cache should receive and deduplicate every loader event.

See [loading-and-ssr.md](references/loading-and-ssr.md).

### Choose the SSR rendering boundary

Create a shared router factory. On the server, pass it and a web-standard
`Request` to `createRequestHandler`; on the client, hydrate with `RouterClient`.
Use:

- `defaultRenderHandler` for the default string-rendered path;
- `renderRouterToString` plus `RouterServer` for custom wrappers or providers;
- `defaultStreamHandler` for automatic streaming and dehydration;
- `renderRouterToStream` plus `RouterServer` for lower-level streaming control.

The handler returns a web-standard `Response`, so framework adapters must
translate request and response objects at their boundary. The standalone React
SSR API is experimental.

See [loading-and-ssr.md](references/loading-and-ssr.md).

### Pick one supported route-splitting shape

For file routes, keep critical options in the normal route file and put render
options in the matching `.lazy.tsx` file with `createLazyFileRoute`. The
`__root` route cannot be split. When a route has no critical options, remove its
empty normal file; the generated tree supplies a virtual anchor.

For code-defined routes, attach a `createLazyRoute` result with `Route.lazy()`.
Use `lazyFn` for a named loader import and usually provide an explicit
`LoaderContext` type.

See
[code-splitting-and-navigation.md](references/code-splitting-and-navigation.md).

## Review checklist

Before finalizing Router or Start changes, verify that:

- shared URLs use `publicHref` when rewrites are configured;
- rewrite composition accounts for reverse output order and automatic basepath
  wrapping;
- mask behavior after reload and sharing is intentional;
- search validators preserve distinct input and output types;
- lazy route files contain only supported render options;
- loader freshness, garbage collection, and stale reload behavior are not being
  conflated;
- SSR loader data stays within serializer limits;
- blocker booleans have not been inverted;
- virtual route punctuation follows virtual rather than physical-file rules;
- build plugins run in the required order and separate plugin instances do not
  share assumed global state.
