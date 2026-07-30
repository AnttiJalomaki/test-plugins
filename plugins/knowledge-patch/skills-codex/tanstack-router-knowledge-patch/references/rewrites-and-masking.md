# URL rewrites and route masks

## Bidirectional URL rewrites

The router-level `rewrite` option separates the public browser URL from the URL
used for route matching:

- `input` maps the browser URL to an internal URL before matching;
- `output` maps an internal URL to a public URL before a link or history entry is
  written.

Each handler receives `{ url: URL }`. It may mutate and return that `URL`, return
a new `URL`, return a full href string, or return `undefined`.

```tsx
import { createRouter } from '@tanstack/react-router'

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

const router = createRouter({
  routeTree,
  rewrite: localeRewrite,
})
```

After matching, `location.href` remains the internal URL and
`location.publicHref` is the external, shareable URL. `<Link>` and programmatic
navigation apply output rewrites automatically. If an output rewrite changes the
origin, `<Link>` performs a hard navigation.

Use the same rewrite configuration while parsing server requests and during SSR
hydration. Otherwise, server and client route matches can disagree.

## Composition and basepaths

`composeRewrites` applies input rewrites from first to last. It applies output
rewrites from last to first so the transformations unwrap in reverse order.

```tsx
import { composeRewrites } from '@tanstack/react-router'

const legacyRewrite = {
  input: ({ url }) => {
    if (url.pathname === '/old') url.pathname = '/new'
    return url
  },
}

const router = createRouter({
  routeTree,
  basepath: '/app',
  rewrite: composeRewrites([localeRewrite, legacyRewrite]),
})
```

A configured `basepath` is automatically composed outside custom rewrites. The
router strips it before custom input handlers run and restores it after custom
output handlers finish. Custom handlers should therefore operate on paths that
do not contain the basepath.

## Route mask APIs

A route mask navigates to one typed runtime location while displaying and
persisting another location in the URL bar. There are two forms:

- supply `mask` to `<Link>` or `navigate()` for one navigation;
- create a typed, reusable mapping with `createRouteMask` and register it in the
  router's `routeMasks` array.

```tsx
import { createRouteMask } from '@tanstack/react-router'

const photoMask = createRouteMask({
  routeTree,
  from: '/photos/$photoId/modal',
  to: '/photos/$photoId',
  params: (prev) => ({ photoId: prev.photoId }),
})

const router = createRouter({
  routeTree,
  routeMasks: [photoMask],
})

navigate({
  to: '/photos/$photoId/modal',
  params: { photoId: 5 },
  mask: {
    to: '/photos/$photoId',
    params: { photoId: 5 },
  },
})
```

## Mask lifetime and reloads

The runtime location behind a mask is stored in browser history state. This has
two consequences:

- copying or sharing the displayed URL loses the state and loads the displayed
  route normally;
- a local reload retains the masked runtime location by default.

Set `unmaskOnReload: true` to discard the hidden runtime location on reload.
Configuration precedence is most-specific first: a per-link or per-navigation
setting overrides the registered route-mask setting, which overrides the router
default.

```tsx
const router = createRouter({
  routeTree,
  routeMasks: [photoMask],
  unmaskOnReload: true,
})
```
