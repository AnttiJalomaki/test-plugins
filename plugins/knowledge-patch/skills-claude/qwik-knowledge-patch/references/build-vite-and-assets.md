# Build, Vite, and Assets

Use this reference for optimizer behavior, Vite configuration, output paths,
preloading, manifests, HMR, integrations, and static generation.

Relevant source batches are `v1.8-1.13`, `v1.14-1.19`,
`v2-alpha-beta-1-9`, `v2-beta-10-19`, `v2-beta-20-29`, and
`v2-beta-30-38`.

## Optimizer ordering and source handling

From 1.8, the optimizer understands `import ... with` and replaces enums with
numbers where possible. Rollup performs QRL grouping, allowing other
Rollup/Vite plugins such as CSS-in-JS transforms to process code before Qwik
transforms it.

A custom `qwikVite()` `fileFilter` cannot exclude `*.qwik.js`, `*.qwik.mjs`,
or `*.qwik.cjs`. These library QRL files are always processed.

The optimizer accepts self-references, including an `AsyncSignal` that writes
to itself. It also understands `@qwik-disable-next-line` as a targeted way to
suppress its next-line diagnostic, such as `preventdefault-passive-check`.

## Experimental Vite flags

`qwikVite()` accepts an `experimental` array. V1 additions included:

- `noSPA` for MPA-only applications that do not use `Link`;
- `valibot` for the experimental `valibot$` validator; and
- `preventNavigate` for asynchronous SPA-navigation blocking through
  `usePreventNavigate`, with browser-dialog fallback for other unsaved-state
  navigation.

```ts
qwikVite({
  experimental: ['noSPA', 'valibot', 'preventNavigate'],
});
```

In later V2 builds, `usePreventNavigate$()` is stable and no longer needs its
flag. Suspense still requires `experimental: ['suspense']`, and background
route loaders require the experimental `blockSSR` feature.

## Asset paths and caching

V1 default build assets changed to `assets/hash-name.ext`; update deploy and
cache rules that assume an older location.

Later V2 builds delegate built-asset placement to Rollup
`output.assetFileNames`. Qwik no longer handles `build.assetsDir`, so configure
the Rollup output when a custom layout is required.

Content-hashed files under `build/` and `assets/` should normally be served
with:

```http
Cache-Control: public, max-age=31536000, immutable
```

Do not apply that rule blindly when Rollup output naming is customized.

## Vite base and output directories

Vite's `base` no longer places the client build under a directory of that
name. The default output remains `dist/`. Choose another path with Vite
`build.outDir` or `qwikVite()`'s `client.outDir` and `ssr.outDir`.

Server chunk files use their own `build/` subdirectory. Client builds now
reject imports of server-only modules instead of silently adding them to the
client graph.

## Client-only and Router development

Core's Vite integration can operate without Qwik Router:

```sh
qwik add csr
```

This configures a client-only application. Router development now uses the
same Node middleware as production and hot-reloads routes added while the
server is running.

## Manifest discovery and artifacts

An SSR `qwikVite()` build automatically reads the client build manifest when
it can. Point it at a nonstandard location with:

```ts
qwikVite({
  ssr: { manifestInputPath: 'dist/client/q-manifest.json' },
});
```

The manifest can still be passed explicitly.

The preloader bundle graph is emitted as an asset. `q-manifest.json` includes
generated assets; by 1.19, `core.js` and `preloader.js` references are filtered
from both the manifest and bundle graph.

Use `getClientManifest()` instead of importing `@qwik-client-manifest`. It is
available through the `@qwik.dev/core` root types. Router `documentHead`
includes the manifest hash for cache busting or ETag generation.

Use the CLI to check that the client output is current:

```sh
qwik check-client
```

## Module preloading and service workers

From 1.14, Qwik uses `modulepreload` links plus a bundle graph containing
dynamic imports and path-to-bundle mappings. Built-in server manifest support
replaces the service workers' prefetch role.

The built-in service-worker components are deprecated. For an uncustomized
worker:

1. Remove `service-worker.ts`.
2. Temporarily retain `ServiceWorkerRegister` so previously deployed workers
   and caches are removed.

Custom worker logic prevents automatic unregistration. Add or retain the
integration only when a legacy application still requires a customizable
worker:

```sh
qwik add service-worker
```

The old service-worker `prefetchStrategy` render option is removed in V2.

Prefetch strategies can set `linkFetchPriority` for generated
`modulepreload` links.

## SSR preload options

SSR preload configuration gained `debug` and stable `maxIdlePreloads`, which
limits concurrent idle preloads:

```ts
renderToStream(<Root />, {
  ...opts,
  preload: { debug: true, maxIdlePreloads: 5 },
});
```

`preloadProbability` is deprecated from 1.16.1. V2 removes
`renderToStream` options `preloads.ssrPreloadProbability`,
`preloads.preloadProbability`, and `preloads.debug`.

## Qwikloader delivery

From 1.15, SSR loads Qwikloader as a separate bundle instead of embedding it.
Tests and unusual network setups can opt back into embedding:

```ts
renderToStream(<Root />, {
  ...opts,
  qwikLoader: 'inline',
});
```

Avoid inline mode as a routine deployment default.

## Vite environment builds

From beta.28, V2 supports Vite's Environment API for building multiple
environments together:

```sh
vite build --app
```

The minimum Vite version for this integration is 6.0.0. Qwik Router adapters
remain an exception: they use another Vite configuration and still require
`build.server` to run separately.

## State-preserving HMR

HMR updates source without losing browser state or forcing a resume.
Development builds therefore send every component's state. Disable HMR when
the extra state exposure or state retention is undesirable:

```ts
qwikVite({ devTools: { hmr: false } });
```

Disabling it restores full-page reloads.

## Static generation

Router SSG runs in a separate worker process. A previously built server output
can run SSG again using its generated script:

```sh
server/run-ssg.js
```

Later V2 builds run SSG in a dedicated `buildApp` Vite environment. Code that
directly calls Vite's programmatic `build()` must change to:

```ts
const builder = await createBuilder(config);
await builder.buildApp();
```

Calling `build()` directly skips prerendering. Fully prerendered, server-free
routes are omitted from the production SSR route plan.

## CLI integrations and defaults

`qwik add` accepts `projectDir`, allowing an integration to target a monorepo
package or subproject:

```sh
qwik add --projectDir=packages/my-package
```

The Tailwind integration supports Tailwind CSS 4, while the CLI still permits
Tailwind CSS 3 projects.

Compiled i18 can be scaffolded directly:

```sh
qwik add compiled-i18
```

The `lint` option defaults to `false`; enable it explicitly when linting is
required.

`isDev`, `isBrowser`, and `isServer` can be imported from the package root. The
older `/build` entry point remains available:

```ts
import { isBrowser, isDev, isServer } from '@builder.io/qwik';
```
