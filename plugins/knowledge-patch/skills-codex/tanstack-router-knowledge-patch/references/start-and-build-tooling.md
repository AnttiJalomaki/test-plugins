# TanStack Start and build tooling

## Deferred hydration boundaries

TanStack Start's compiler can code-split `Hydrate` boundaries. The integration:

- preloads the generated client chunks;
- preserves server-rendered fallback HTML until hydration;
- replays interactions that triggered hydration after the boundary becomes
  interactive;
- works with both Vite and Rsbuild.

## Hydration before initial matching

Custom router hydration runs before the first client route match. Install
hydrated configuration, such as request-specific URL rewrites, during that phase
so it is present before SSR hydration compares route matches.

## Runtime SSR asset transformation

TanStack Start SSR supports inline CSS manifests that hydrate without adding
duplicate stylesheet links. `transformAssets` also supports runtime-configurable
inline CSS and opt-in CSS URL templates.

For Rsbuild client output, module scripts are the default and IIFE output is
available for classic-script environments. A `transformAssets` script callback
receives only:

```ts
{ kind: 'script', url }
```

Configure cross-origin behavior for script assets with the `script` key.

## React Server Component imports

The `@tanstack/react-router` package root has a `react-server` export condition.
It preserves the usual API surface while resolving `notFound` and `redirect`
from a server-safe entry. React Server Components and server functions can keep
using package-root imports:

```tsx
import { notFound, redirect } from '@tanstack/react-router'
```

## Intent tooling

`@tanstack/intent` supplies agent-oriented skills and CLI entry points for
TanStack Router and TanStack Start packages.

## Virtual route punctuation

The route generator treats punctuation differently in explicit virtual config
and physical file routes:

- dots in explicit virtual route paths and pathless layout IDs are preserved;
- leading and trailing underscores in virtual `route()` paths are literal URL
  characters;
- physical file routes continue to use bracket escapes for literal underscore
  segments.

The physical-file escape rule also applies to index routes beneath pathless
layouts, `physical()` prefixes, and `__virtual.ts` subtrees.

## Virtual config imports and tokens

Virtual route config files may import through aliases defined in `tsconfig`; the
route generator resolves those aliases while loading the config.

Custom `routeToken` and `indexToken` values may begin with regex metacharacters,
including `+`. The generator treats the configured token as a token rather than
accidentally interpreting its first character as regex syntax.

## Plain TypeScript transform parsing

When a filename is available, router and Start import-protection transforms
parse plain TypeScript files without JSX. Angle-bracket type assertions are
therefore interpreted as TypeScript rather than misread as JSX.

## Isolated plugin instances

Every router plugin instance carries explicit context rather than using global
route metadata. Multiple plugin instances can process separate route sets
without cross-transforming each other's route files.

## Supported build integrations

`@tanstack/router-plugin` supports Rsbuild, accepts Vite 8 as a peer, and
supports `vite-plugin-solid` starting at `3.0.0-0`.

## Expanded React route HMR

React route hot-module replacement preserves state for auto-split components
and lowercase-named component functions. Development transforms cover split
component groups and the unsplit root shell, pending, and error options.

Aliased route imports retain generated properties during HMR transforms.
`createRootRouteWithContext` calls with type arguments are recognized by Vite
Fast Refresh. Webpack and Rspack route HMR no longer import the optional
`react-refresh/runtime` package.
