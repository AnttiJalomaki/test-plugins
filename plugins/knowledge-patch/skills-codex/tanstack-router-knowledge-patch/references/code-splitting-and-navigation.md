# Code splitting and navigation control

## Route-directory encapsulation

A file route can move from `posts.tsx` to `posts/route.tsx` without extra
configuration. Use the directory form to colocate the route with its lazy
render files and other route-specific modules.

## Automatic file-route splitting

`autoCodeSplitting` belongs to the bundler plugin. It does not work with
`@tanstack/router-cli` by itself.

```ts
plugins: [
  tanstackRouter({ autoCodeSplitting: true }),
  react(),
]
```

The framework plugin must come after the router plugin. Automatic splitting
extracts only:

- `component`;
- `errorComponent`;
- `pendingComponent`;
- `notFoundComponent`.

Loaders, `beforeLoad`, search validation, context, static data, links, scripts,
styles, and all other matching configuration remain in the critical route
chunk.

## Manual lazy file boundaries

Without automatic splitting, keep critical options in the normal route file.
Put the four supported render options in a matching `.lazy.tsx` file created
with `createLazyFileRoute`.

```tsx
// routes/posts.tsx
export const Route = createFileRoute('/posts')({ loader: fetchPosts })

// routes/posts.lazy.tsx
export const Route = createLazyFileRoute('/posts')({ component: Posts })
```

The `__root` route cannot be split. If a route has no critical configuration,
delete its empty normal file; the generated route tree supplies a virtual anchor
for its lazy file.

## Code-defined routes and lazy loaders

For code-defined routes, create a lazy render module with `createLazyRoute` and
attach it with `Route.lazy()`:

```tsx
// posts.lazy.tsx
export const Route = createLazyRoute('/posts')({ component: Posts })

const postsRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/posts',
}).lazy(() => import('./posts.lazy').then((mod) => mod.Route))
```

Use `lazyFn` to import a named loader separately:

```tsx
const dataRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/report',
  loader: lazyFn(() => import('./loader'), 'loader'),
})
```

The loader module generally needs an explicit `LoaderContext` type. File-based
loaders cannot use this manual `lazyFn` pattern; split them only through
automatic splitting with customized bundling options.

## Resolver-based navigation blocking

`useBlocker` supplies typed `current` and `next` locations to `shouldBlockFn`.
Returning `true` blocks navigation. With `withResolver: true`, the blocker
enters a pending blocked state and waits for the application to resolve it:

```tsx
const { status, proceed, reset } = useBlocker({
  shouldBlockFn: ({ current, next }) => formIsDirty,
  withResolver: true,
  enableBeforeUnload: formIsDirty,
})

if (status === 'blocked') {
  // “Leave” calls proceed(); “Stay” calls reset().
}
```

`proceed()` permits the navigation and `reset()` cancels it. The
`enableBeforeUnload` option separately controls the browser-native prompt for a
reload or tab close; resolver mode does not replace that setting.

## Asynchronous blocker decisions

Without resolver mode, `shouldBlockFn` may return a promise while custom UI asks
for a decision. Resolve `true` to cancel navigation and `false` to allow it.

```tsx
useBlocker({
  shouldBlockFn: () =>
    formIsDirty ? askWhetherToLeave().then((leave) => !leave) : false,
})
```

Take care when adapting a dialog whose result is phrased as “leave?” because its
boolean must be inverted to answer the blocker's “block?” question.
