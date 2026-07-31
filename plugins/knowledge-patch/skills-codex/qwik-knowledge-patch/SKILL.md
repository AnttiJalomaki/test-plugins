---
name: qwik-knowledge-patch
description: Qwik
version: 1.19.0
license: MIT
metadata:
  author: Nevaberry
---



# Qwik Knowledge Patch

Use this skill when maintaining Qwik applications, Qwik City routing, Qwik
libraries, SSR integrations, or their Vite build configuration. Start with the
quick references below, then open the topic file that matches the task.

## When to load this skill

Load it when:

- upgrading an existing Qwik application or library;
- diagnosing optimizer, Vite, preload, manifest, or asset-output changes;
- updating reactive state, tasks, computed values, or store access;
- changing Qwik City routing, navigation, request handling, or caching;
- working with MDX, error boundaries, events, or route-data tests; or
- reviewing deprecated APIs before a later migration.

Prefer the project manifest, lockfile, configuration, code, and tests when they
show behavior that differs from this guidance.

## Reference index

| Reference | Topics |
| --- | --- |
| [Async reactivity and state](references/async-reactivity-and-state.md) | Async computation, tasks, stores, tracking, and notification behavior |
| [Build and deployment](references/build-and-deployment.md) | Optimizer, Vite, assets, preload, manifests, service workers, CLI, and integrations |
| [Components and events](references/components-and-events.md) | MDX, error boundaries, view-transition events, and route-data testing |
| [Migration and packaging](references/migration-and-packaging.md) | Library publishing, mixed-generation consumers, dependency placement, and exports |
| [Router and navigation](references/router-and-navigation.md) | SPA prevention, previous URLs, rewrites, redirects, and cache behavior |
| [Server and route data](references/server-and-route-data.md) | Server-function errors, request events, origins, redirect responses, and rewrites |

## Breaking-change triage

### Update asset paths and cache rules

Default build assets use `assets/hash-name.ext`. Replace deployment rules,
CDN patterns, or CSP generation that assumes an older location. Apply
long-lived immutable caching only to content-hashed files under `build/` and
`assets/`, unless Rollup output names were customized.

See [Build and deployment](references/build-and-deployment.md#asset-paths-and-cache-headers).

### Keep library QRL files in the transform

A custom `qwikVite()` `fileFilter` cannot exclude `*.qwik.js`,
`*.qwik.mjs`, or `*.qwik.cjs`. These library QRL files are always processed.
Do not use the filter as a way to suppress their Qwik transform.

See [Build and deployment](references/build-and-deployment.md#library-qrl-file-filtering).

### Make Vite an application dependency

Vite is a peer dependency of Qwik, Qwik City, Qwik React, and Qwik Labs.
Declare Vite directly in the application so package resolution does not create
duplicate Vite imports. Qwik core and Qwik City projects using the newer
toolchain must use Vite 7.

See [Migration and packaging](references/migration-and-packaging.md#vite-dependency-placement)
and [Build and deployment](references/build-and-deployment.md#vite-7).

### Rework library publishing

Qwik library builds no longer run the Qwik transform. Publish a fresh library
build and expand the accepted Qwik range when the package must support both
generations. A later-generation project can retain a first-generation library
by installing both runtimes as described in the packaging reference.

See [Migration and packaging](references/migration-and-packaging.md#library-builds-and-mixed-generation-consumers).

### Migrate built-in service-worker prefetching

Qwik now preloads bundles with `modulepreload` links and a bundle graph.
Built-in service-worker components are deprecated. For an uncustomized worker,
remove `service-worker.ts` but temporarily keep `ServiceWorkerRegister` so
deployed workers and caches are removed. Preserve the integration when custom
worker logic prevents automatic unregistration.

See [Build and deployment](references/build-and-deployment.md#automatic-preloading-and-service-worker-migration).

## Deprecation quick reference

### Do not use async callbacks in `useComputed$`

Async `useComputed$` callbacks fail to track signals first read after an
`await`, and an initial promise restarts rendering. Move asynchronous work to
`useTask$` or `useResource$`.

See [Async reactivity and state](references/async-reactivity-and-state.md#async-computed-functions).

### Remove task eagerness

The `eagerness` option of `useTask$` is deprecated and should not be used in
new code.

See [Async reactivity and state](references/async-reactivity-and-state.md#usetask-eagerness).

### Replace `preloadProbability`

`preloadProbability` is deprecated. Use the supported preload controls,
including `maxIdlePreloads` for concurrent idle-preload limits.

See [Build and deployment](references/build-and-deployment.md#ssr-preload-configuration).

## Configuration quick reference

### Gate experimental features explicitly

Pass experimental feature names through the `qwikVite()` `experimental`
array:

```ts
qwikVite({
  experimental: ['noSPA', 'valibot', 'preventNavigate'],
});
```

Use `noSPA` only for MPA-only applications that do not use `Link`,
`valibot` for `valibot$` validation, and `preventNavigate` for
`usePreventNavigate`.

### Tune SSR preloading

```ts
renderToStream(<Root />, {
  ...opts,
  preload: {
    debug: true,
    maxIdlePreloads: 5,
  },
});
```

`maxIdlePreloads` is the stable concurrent idle-preload limit. Prefetch
strategies can also set `linkFetchPriority` for generated `modulepreload`
links.

### Inline the Qwikloader only when necessary

SSR normally loads the Qwikloader from a separate bundle. Tests and unusual
network setups can opt back into embedding it:

```ts
renderToStream(<Root />, {
  ...opts,
  qwikLoader: 'inline',
});
```

## Runtime and reactivity quick reference

### Extract raw store data for platform APIs

Use `unwrapStore()` before passing store content to structured cloning or
IndexedDB:

```ts
const copy = structuredClone(unwrapStore(store));
```

### Account for membership tracking

`"prop" in store` creates a reactive subscription. Consumers that execute
membership checks rerun when the property's presence changes.

### Read without tracking

`untrack()` accepts signals and stores directly. Its callback form accepts
arguments:

```ts
const value = untrack(signal);
const result = untrack((a, b) => a + b, 1, 2);
```

Computed signals notify listeners only when the computed result changes, not
merely when a dependency changes.

## Router and server quick reference

### Block navigation with the matching experiment

`usePreventNavigate` asynchronously blocks SPA navigation. Other unsaved-state
navigation falls back to browser dialogs. Enable `preventNavigate` before
using it.

### Throw rewrites from request handlers

`RequestEvent.rewrite()` performs an internal redirect while preserving the
browser-visible URL:

```ts
export const onRequest: RequestHandler = async ({ rewrite }) => {
  throw rewrite('/articles/42');
};
```

Multiple rewrite routes may target the same destination. On the first render,
the router's previous URL is `undefined`, so consumers must handle its
absence.

### Catch server-function failures in middleware

Errors are standardized across `server$` functions and route loaders.
`@plugin` middleware can catch `server$` failures. Client calls throw for 4xx
statuses and statuses above 500, while 499 is accepted.

See [Server and route data](references/server-and-route-data.md#server-function-error-flow).

## Components, CLI, and testing quick reference

Imported MDX accepts a `components` prop, can use props in JavaScript
expressions, and honors default-exported layout components. Use
`ErrorBoundary` for component error handling; corrected
`useErrorBoundary` behavior is also available.

Qwik emits `qviewTransition` when a view transition starts. Listen for that
`CustomEvent` when application code needs transition lifecycle behavior.

For monorepos, target a package with:

```sh
qwik add --projectDir=packages/my-package
```

Use `QwikCityMockProvider` to mock route loaders and actions. Run
`qwik check-client` when validating that a client bundle is fresh.

## Verification checklist

Before completing a change:

- inspect `package.json` and the lockfile for direct Vite ownership;
- verify generated asset, manifest, loader, and preloader paths;
- test navigation with an absent initial previous URL and with redirects;
- exercise client-visible server failures and middleware interception;
- check reactive consumers after changing `untrack()` or store membership;
- test deployed service-worker cleanup before removing registration; and
- confirm cache headers distinguish hashed assets, route data, and redirects.
