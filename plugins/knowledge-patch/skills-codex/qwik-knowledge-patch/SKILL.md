---
name: qwik-knowledge-patch
description: Qwik
version: 2.0.0-beta.38
license: MIT
metadata:
  author: Nevaberry
---


# Qwik

Use this skill when maintaining, migrating, building, deploying, or testing Qwik
applications and libraries. Start with the breaking-change checks below, then
open the topic reference that matches the work.

## Reference index

| Reference | Topics |
| --- | --- |
| [migration-and-packaging.md](references/migration-and-packaging.md) | V2 migration, package renames, ESM, third-party libraries, types, router roots, serialized state |
| [build-and-deployment.md](references/build-and-deployment.md) | Optimizer and Vite configuration, assets, manifests, preload behavior, HMR, adapters, SSG |
| [async-reactivity-and-state.md](references/async-reactivity-and-state.md) | Async computation, signals, task cleanup, stores, serialization, `Show`, Suspense |
| [components-and-events.md](references/components-and-events.md) | MDX, JSX attributes and events, bindings, error boundaries, loader events, testing |
| [router-and-navigation.md](references/router-and-navigation.md) | Router setup, navigation, rewrites, layouts, document head, error routes, middleware |
| [server-and-route-data.md](references/server-and-route-data.md) | Server functions, request events, loaders, caching, ETags, SSR and request origins |

## Breaking changes and migration checks

### Run the migration before hand-editing

```sh
pnpm qwik migrate-v2
```

The migration rewrites package names, renamed identifiers, Qwik and Vite
configuration, and dependencies. After it runs, verify the changes that require
project-specific judgment:

- Use `@qwik.dev/core`, `@qwik.dev/router`, and `@qwik.dev/react` in place of
  their V1 packages.
- Use `@qwik.dev/core/jsx-runtime` for the JSX runtime.
- Replace `@qwik-city-plan` with `@qwik-router-config` where applicable.
- Set TypeScript `moduleResolution` to `Bundler` and `jsxImportSource` to
  `@qwik.dev/core`.
- Make the package ESM with `"type": "module"`.
- Keep `@qwik.dev/*` packages in `devDependencies`.

Core and Router are ESM-only. Update CommonJS consumers and build tools instead
of expecting CJS or UMD artifacts.

### Prevent duplicate runtimes from legacy libraries

Third-party libraries that still depend on V1 package names can install a
second runtime. Override those names to V2, bundle the library in SSR with
`ssr.noExternal`, and exclude it from `optimizeDeps`. See
[migration-and-packaging.md](references/migration-and-packaging.md) for the
complete configuration.

V1-era library publishers should publish a fresh library build and extend the
accepted Qwik range to include V2. During a staged consumer migration, both
runtime generations can be installed intentionally, but a V2 page containing
V1 containers must load both generations of the qwikloader.

### Use the current async-computation API

Use async `useComputed$()` for new V2 code. `useAsync$()`, `createAsync$()`,
and `AsyncSignal` are deprecated. Read failures from `.error`; reading `.value`
after a failure rethrows. Use the compute context's `track()` for dependencies
first read after an `await`, and use its `abortSignal` for cancellation.

When migrating `useResource$()` or `<Resource />`, account for these differences:

- `.value` is `T`, not `Promise<T>`; unresolved reads throw.
- `.pending` reports unresolved state and `.error` reports failure.
- `initial`, `previous`, `expires`, and `poll` cover initialization and refresh.
- `concurrency: 0` preserves unlimited resource-style parallelism.
- A legacy `useAsync$().promise()` returns `Promise<void>`, not the result.

Do not use an async `useComputed$()` callback in a V1 application: only signal
reads before its first `await` are tracked there. Use `useTask$()` or
`useResource$()` until the application migrates.

### Remove retired and renamed options

- Remove `useVisibleTask$({ eagerness: ... })` in V2.
- Do not add new uses of `useTask$`'s deprecated `eagerness` option.
- Replace `Link`'s `prefetchBundle` with `prefetchBundles`.
- Replace legacy `useAsync$` option `interval` with `expires`; use `poll` to
  schedule reruns after expiration.
- Remove `renderToStream` preload probability and debug options that V2 no
  longer accepts.
- Remove the deprecated service-worker `prefetchStrategy`.
- Do not configure removed `qwikVite()` options such as `symbolMapper` or
  deprecated `srcInput`.

### Move optimizer and asset configuration

Import optimizer bindings from `@qwik.dev/optimizer`; the compatibility export
at `@qwik.dev/core/optimizer` remains available, while the Vite plugin stays in
core. Configure built-asset placement with Rollup `output.assetFileNames`.
Qwik no longer owns `build.assetsDir`, and default hashed assets use the
`assets/hash-name.ext` form.

## Build and deployment quick reference

### Match the Vite integration to the build

Applications must depend on Vite directly because it is a peer dependency.
Modern V2 application builds can use Vite's Environment API:

```sh
vite build --app
```

Router adapters remain a separate-build exception. For programmatic SSG, call
`createBuilder().buildApp()`; a direct `build()` call skips prerendering. Use
`client.outDir`, `ssr.outDir`, or Vite `build.outDir` for output placement;
Vite `base` does not relocate the client output.

If automatic SSR manifest discovery cannot find a custom client manifest, set:

```ts
qwikVite({
  ssr: { manifestInputPath: 'dist/client/q-manifest.json' },
});
```

### Treat preloading and the service worker separately

Bundle preloading uses `modulepreload`, a generated bundle graph, and built-in
server manifest support. The old built-in prefetch service-worker components
are deprecated. For an uncustomized worker, remove `service-worker.ts` but keep
`ServiceWorkerRegister` temporarily so deployed workers and caches are removed.
Add the service-worker integration only for genuinely custom worker logic.

Serve content-hashed files under `build/` and `assets/` with long-lived
immutable caching unless output names are customized. Follow explicit cache
headers for route data instead of assuming it must always be fresh.

### Keep server-only code out of the client graph

Client builds reject server-only imports. If a server module is legitimately
needed in server output, configure `ssr.noExternal`; the build reports missing
externalization configuration rather than scanning every module up front.

## Router and server quick reference

### Use current navigation semantics

SPA view transitions are opt-in on `QwikRouterProvider`. `usePreventNavigate$`
and `request.rewrite()` are stable without experimental flags. Returning
`redirect()`, `error()`, or `rewrite()` from loaders, actions, handlers, and
server functions has the same control-flow effect as throwing it.

On the initial render, the previous URL is `undefined`. For internal rewrites,
the visible URL is preserved; invalid absolute rewrite URLs produce `400`.
Multiple rewrite routes may target the same destination.

### Declare loader inputs and cache behavior

Strict loaders are enabled by default. Declare query keys with `search`; without
it, the loader receives no query parameters and does not rerun for query-string
changes. A loader cannot read action state, so read that signal in the component
or put relevant state in the URL.

Route-loader failures appear in `.error`, and `.value` rethrows; `fail()` instead
stores `{ failed }`. Configure `expires`, `poll`, `allowStale`, `eTag`, and
`cacheKey` deliberately. The transport uses manifest-versioned loader JSON and
defaults expiry to two minutes. Treat private or user-specific results as
short-lived and validate them with ETags.

### Configure error routes and layouts

Use `404.tsx` for misses and `error.tsx` for other status codes. Both render in
the layout selected by `@layout` and `!` modifiers; rename the not-found route
to `404!.tsx` when it must bypass layouts. The nearest not-found route wins.

## Reactivity and event quick reference

Use `signal.untrackedValue` for reads or writes that must not subscribe, then
call `signal.trigger()` after an in-place mutation or untracked write when
subscribers need to run. `untrack()` also accepts a signal, a store, or a
callback plus arguments.

Event names without a leading dash are lowercased; a leading dash preserves
case, while later dashes remain dashes. Custom-event segments in
`preventdefault:event` and `stoppropagation:event` must be kebab-case. Passive
and capture markers pair with handlers, such as `passive:touchmove` with
`onTouchMove$`.

`useTask$()` and `useVisibleTask$()` await a returned cleanup promise before
their next invocation. Do not return that promise when overlap is intentional.
Rendering does not wait for visible-task callbacks, which run as post-flush
effects.

## Verification checklist

Before finalizing a change:

1. Confirm package names, ESM mode, TypeScript settings, and dependency section.
2. Check that only one intended Qwik runtime reaches each page.
3. Verify client and server output directories and manifest discovery.
4. Exercise async pending, failure, cancellation, and concurrency paths.
5. Test strict-loader query keys, expiry, ETags, and user-specific caching.
6. Test `404.tsx` and `error.tsx` inside the intended layout hierarchy.
7. Confirm event-name casing and passive/default-prevention combinations.
8. Run SSG through the app builder and verify immutable asset cache headers.
