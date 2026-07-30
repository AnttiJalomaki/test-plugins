# Build and deployment

## Optimizer behavior

The optimizer understands import attributes written as `import ... with` and
replaces enums with numeric values where possible. QRL grouping is delegated to
Rollup, so CSS-in-JS and other Rollup/Vite plugins can process code before the
Qwik transform.

A custom `qwikVite()` `fileFilter` cannot exclude library QRL files named
`*.qwik.js`, `*.qwik.mjs`, or `*.qwik.cjs`; those files are always processed.
The optimizer also accepts a symbol that refers to its own `AsyncSignal`.

Optimizer bindings are packaged in `@qwik.dev/optimizer` in V2. The
`@qwik.dev/core/optimizer` path re-exports the runtime for compatibility, and
the Vite plugin remains bundled with core.

Use `@qwik-disable-next-line` to suppress one optimizer diagnostic on the next
line, for example `preventdefault-passive-check` when its event analysis is not
appropriate.

## Vite plugin options and dependencies

Vite is a peer dependency of Qwik, Qwik City/Router, Qwik React, and Qwik Labs.
The application must declare a direct Vite dependency to avoid duplicate
imports.

The `qwikVite()` `experimental` array introduced flags for:

- `noSPA`, for MPA-only applications that do not use `Link`;
- `valibot`, for the experimental `valibot$` validator;
- `preventNavigate`, for the original experimental form of
  `usePreventNavigate`.

```ts
qwikVite({ experimental: ['noSPA', 'valibot'] });
```

The navigation-prevention API later became stable as `usePreventNavigate$`, so
new V2 configuration does not require its old experimental flag.

Core now handles development QRL segment mapping, so no separate
`symbolMapper` function is needed. The unused `srcInput` option is deprecated.
The `lint` option defaults to `false`; enable it explicitly when the build is
expected to lint.

Qwik core and Qwik City applications on 1.16 moved to Vite 7. V2 Environment
API support requires at least Vite 6.0.0, and migrated V2 projects can use Vite
8 and Rolldown. Select the Vite version for the exact Qwik generation and build
path rather than assuming one minimum applies to all of them.

## CLI integrations

Target an integration at a monorepo package with `projectDir`:

```sh
qwik add --projectDir=packages/my-package
```

The CLI can scaffold a client-only application or compiled internationalization:

```sh
qwik add csr
qwik add compiled-i18
```

Client-only mode lets the core Vite integration operate without Qwik Router.
Use `qwik check-client` to verify that the client bundle is fresh.

Tailwind integration supports Tailwind CSS 4, and the CLI also permits a
project to remain on Tailwind CSS 3.

## Asset paths and cache headers

Default built assets use `assets/hash-name.ext`; update deployment and caching
rules that expect the earlier placement. In V2, configure placement with
Rollup's `output.assetFileNames`. Qwik no longer handles Vite's
`build.assetsDir` option on its behalf.

Serve content-hashed files beneath `build/` and `assets/` with this policy
unless Rollup output naming has been customized:

```http
Cache-Control: public, max-age=31536000, immutable
```

Qwik City navigation honors the cache headers on `q-data.json` rather than
forcing a fresh download. Its default cache duration in the V1 flow is one
hour.

## Bundle preloading and service workers

From 1.14, Qwik preloads with `modulepreload` links and a bundle graph that
contains dynamic imports and path-to-bundle mappings. The server has built-in
manifest support, so the built-in prefetch service worker is no longer the
normal preload mechanism.

For an uncustomized service worker, remove `service-worker.ts` but keep
`ServiceWorkerRegister` temporarily. This lets deployed workers unregister and
their caches be removed. Custom worker code prevents automatic unregistration;
add the integration only when that logic remains necessary:

```sh
qwik add service-worker
```

The V1 SSR preload configuration accepts `debug` and stable
`maxIdlePreloads`, which limits concurrent idle preloads:

```ts
renderToStream(<Root />, {
  ...opts,
  preload: { debug: true, maxIdlePreloads: 5 },
});
```

`preloadProbability` is deprecated from 1.16.1. V2 removes
`preloads.ssrPreloadProbability`, `preloads.preloadProbability`, and
`preloads.debug`, as well as the deprecated service-worker
`prefetchStrategy`.

Prefetch strategies that still generate `modulepreload` links can set
`linkFetchPriority`.

## Qwikloader and manifest artifacts

From 1.15, SSR loads the Qwikloader as a separate bundle rather than embedding
it. Tests or unusual network setups can opt into the inline form from 1.17:

```ts
renderToStream(<Root />, { ...opts, qwikLoader: 'inline' });
```

The preloader bundle graph is emitted as an asset, and `q-manifest.json`
includes generated assets. By 1.19, `core.js` and `preloader.js` references are
filtered out of the manifest and bundle graph.

In V2 code, call `getClientManifest()` instead of importing
`@qwik-client-manifest`; it is typed from the `@qwik.dev/core` root. Router
`documentHead` includes the manifest hash for cache busting or ETag generation.

## Client and server output

`qwikVite()` SSR builds discover the client build manifest when possible. If
`q-manifest.json` is elsewhere, set its path, or continue passing a manifest
explicitly:

```ts
qwikVite({
  ssr: { manifestInputPath: 'dist/client/q-manifest.json' },
});
```

Vite `base` no longer nests the client build below that path; the default
output stays `dist/`. Choose a different location with Vite `build.outDir` or
the Qwik plugin's `client.outDir` and `ssr.outDir`.

Server chunks live in a dedicated `build/` subdirectory. Forked router adapters
must remove imports of `@qwik-router-not-found-paths` and
`@qwik-router-static-paths`, because that metadata is embedded directly.

Qwik no longer scans every module at build startup to decide what belongs in
server output. A faster build-time check reports when `ssr.noExternal` must be
updated. Client builds reject server-only modules that reach the client graph.

## Environment builds and HMR

From V2 beta.28, Vite's Environment API can build multiple environments in one
application build:

```sh
vite build --app
```

Router adapters are the exception: they use another Vite configuration and
still require `build.server` to run separately.

Development HMR updates browser code without losing state or forcing a resume.
To accomplish this, development sends every component's state. Restore
full-page reload behavior when that cost is undesirable:

```ts
qwikVite({ devTools: { hmr: false } });
```

Router development uses the same Node middleware as production and hot-reloads
newly created routes.

## SSG and repeatable builds

Router SSG runs in a separate worker process. After server output has already
been built, repeat SSG through its generated script:

```sh
server/run-ssg.js
```

V2 SSG is a dedicated Vite build environment under `buildApp`. Code that calls
Vite programmatically must use:

```ts
const builder = await createBuilder(config);
await builder.buildApp();
```

Calling the lower-level `build()` directly skips prerendering. Fully
prerendered routes that need no server are omitted from the production SSR
route plan.
