# URL Rewrites, Route Masks, and Navigation Blocking

## Bidirectional URL rewrites

Configure the router-level `rewrite` option to separate the public URL from the
internal URL used for matching:

- `input` maps the browser or request URL to an internal URL before matching.
- `output` maps an internal destination back before Router writes a link or
  browser history entry.

Each function receives `{ url: URL }`. It may return the mutated object, a new
`URL`, a complete href string, or `undefined`.

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

`location.href` remains the internal value. Use `location.publicHref` for the
external, shareable value.

`<Link>` and programmatic navigation automatically apply output rewrites. If an
output changes the origin, `<Link>` uses a hard navigation. Server request
parsing and server-rendering hydration use the same rewrite configuration, so
install any request-specific rewrite state before initial client matching.

## Composition and basepaths

`composeRewrites` applies input functions from first to last, then output
functions from last to first. The inverse output order unwraps nested
transformations correctly.

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

A configured `basepath` is automatically composed outside the custom rewrite:
Router removes it before custom input and restores it after custom output. Do
not manually duplicate the basepath inside custom handlers.

## Route masking

A mask navigates to one typed runtime location while displaying and persisting
a different location in the address bar.

For one navigation, pass `mask` to `<Link>` or `navigate()`. For reusable,
typed mappings, create a mask with `createRouteMask` and register it under the
router's `routeMasks`.

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

The runtime location is stored in browser history state rather than encoded in
the displayed URL. Copying or sharing that URL loses the hidden location and
loads the displayed route normally.

A reload in the same browser history entry preserves the mask by default. Set
`unmaskOnReload: true` to discard it:

```tsx
const router = createRouter({
  routeTree,
  routeMasks: [photoMask],
  unmaskOnReload: true,
})
```

The most local setting wins: a link or navigation setting overrides a
route-mask setting, which overrides the router default.

## Resolver-based navigation blocking

`useBlocker` supplies typed `current` and `next` locations to
`shouldBlockFn`. Returning true blocks the navigation.

With `withResolver: true`, a block enters a pending blocked state instead of
settling immediately. Call `proceed` to continue or `reset` to remain:

```tsx
const { status, proceed, reset } = useBlocker({
  shouldBlockFn: () => formIsDirty,
  withResolver: true,
  enableBeforeUnload: formIsDirty,
})

if (status === 'blocked') {
  // Wire proceed() to “Leave” and reset() to “Stay”.
}
```

`enableBeforeUnload` is independent from client navigation blocking. It
controls the browser's native prompt for reloads and closing the tab.

## Asynchronous blocker decisions

Without resolver mode, `shouldBlockFn` may return a promise for custom
asynchronous UI. Its boolean meaning follows blocking, not leaving: resolve
`true` to cancel navigation and `false` to allow it.

```tsx
useBlocker({
  shouldBlockFn: () =>
    formIsDirty ? askWhetherToLeave().then((leave) => !leave) : false,
})
```
