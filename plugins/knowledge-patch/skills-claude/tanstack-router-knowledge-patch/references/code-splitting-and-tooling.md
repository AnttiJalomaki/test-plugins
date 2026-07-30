# Code Splitting, Route Generation, and Build Tooling

## Route-directory encapsulation

A file route can move from a flat file such as `posts.tsx` to
`posts/route.tsx` without extra configuration. This makes it possible to
colocate the eager route module with its split components and related files.

## Automatic file-route splitting

`autoCodeSplitting` is implemented by a bundler plugin. It is not available
from `@tanstack/router-cli` alone. Register the router plugin before the
framework plugin:

```ts
plugins: [
  tanstackRouter({ autoCodeSplitting: true }),
  react(),
]
```

Automatic splitting lazily extracts only these render options:

- `component`
- `errorComponent`
- `pendingComponent`
- `notFoundComponent`

Loaders, `beforeLoad`, search validation, context, static data, links, scripts,
styles, and all other matching configuration stay in the critical chunk.

## Manual file-route lazy boundaries

Without automatic splitting, keep critical options in the normal file and put
the four supported render options in a same-path `.lazy.tsx` module created
with `createLazyFileRoute`.

```tsx
// routes/posts.tsx
export const Route = createFileRoute('/posts')({ loader: fetchPosts })

// routes/posts.lazy.tsx
export const Route = createLazyFileRoute('/posts')({ component: Posts })
```

The `__root` route cannot be split. When a route has no critical
configuration, delete its empty normal file; the generated route tree supplies
a virtual eager anchor for the lazy file.

## Code-defined routes and split loaders

Code-defined routes attach a `createLazyRoute` export with `Route.lazy()`:

```tsx
// posts.lazy.tsx
export const Route = createLazyRoute('/posts')({ component: Posts })

const postsRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/posts',
}).lazy(() => import('./posts.lazy').then((mod) => mod.Route))
```

A code-route loader can be imported by export name through `lazyFn`:

```tsx
const dataRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/report',
  loader: lazyFn(() => import('./loader'), 'loader'),
})
```

The lazy loader's context generally needs an explicit `LoaderContext` type.
File-based loaders cannot use this manual pattern; split them only through
automatic splitting with customized bundling options.

## Virtual-route punctuation

The route generator preserves dots in explicit virtual route paths and
pathless layout IDs. It no longer treats those dots as flat-file separators.

Leading and trailing underscores in virtual `route()` paths are literal URL
characters. Physical file routes still require bracket escapes for literal
underscore segments. This remains true for:

- index routes under pathless layouts
- `physical()` prefixes
- `__virtual.ts` subtrees

Keep the virtual and physical punctuation rules separate during migrations.

## TypeScript aliases and parsing

Virtual route configuration files can import through aliases declared in
`tsconfig`; the generator resolves them while loading the configuration.

When the transform has a filename, Router and Start import-protection
transforms parse plain TypeScript without JSX. Angle-bracket type assertions in
`.ts` files are therefore not interpreted as JSX.

## Custom route tokens

File-route generation accepts custom `routeToken` and `indexToken` strings that
begin with regular-expression metacharacters such as `+`. Do not pre-escape or
reject such tokens solely because they are meaningful in regular expressions.

## Isolated plugin instances

Each router plugin instance owns explicit context rather than sharing global
route metadata. Multiple instances should not cross-transform one another's
route files. When configuring several route trees, still give each plugin a
clear include scope and verify its generated output.

## Rsbuild and peer compatibility

`@tanstack/router-plugin` supports Rsbuild. Its accepted peers include Vite 8
and `vite-plugin-solid` beginning at `3.0.0-0`.

For Rsbuild client assets, module scripts are the default; select IIFE output
for classic-script environments. See the runtime asset details in
`loaders-and-ssr.md`.

## Expanded route HMR

React route hot replacement preserves component state for auto-split
components and lowercase-named functions. Development transforms handle:

- split component groups
- unsplit root shell options
- unsplit pending and error options

Aliased route imports retain generated properties. Vite Fast Refresh recognizes
`createRootRouteWithContext` calls that include type arguments. Webpack and
Rspack route HMR no longer import the optional `react-refresh/runtime` package.

Test state preservation with the actual transformed route shape rather than
assuming a function's capitalization or split status excludes it.

## Intent tooling

`@tanstack/intent` supplies agent-oriented skills and CLI entry points for
TanStack Router and TanStack Start packages.
