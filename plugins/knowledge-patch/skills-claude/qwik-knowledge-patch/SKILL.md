---
name: qwik-knowledge-patch
description: Qwik
version: 1.19.0
license: MIT
metadata:
  author: Nevaberry
---


# Qwik Development Guide

Use this skill for compatibility-sensitive Qwik application, library, Qwik City,
optimizer, Vite, SSR, routing, and reactivity work.

## Operating method

1. Inspect the project's Qwik, Qwik City, and Vite versions before changing code.
2. Identify whether the task concerns build output, package migration, reactivity,
   component authoring, or router/server behavior.
3. Read the matching reference file from the index below.
4. Apply the breaking-change and deprecation guidance before adopting newer APIs.
5. Preserve deployment behavior explicitly: asset paths, cache headers, preload
   behavior, and service-worker cleanup can all affect existing installations.

## Reference index

| Reference | Topics |
| --- | --- |
| [Build, Vite, and assets](references/build-vite-and-assets.md) | Optimizer transforms, asset paths, QRL filtering, experimental flags, preloading, service workers, cache headers, manifests, Qwikloader, Vite, and CLI checks |
| [JSX, events, and serialization](references/jsx-events-and-serialization.md) | Raw stores, MDX components, view-transition events, and error boundaries |
| [Migration and packages](references/migration-and-packages.md) | Library publishing, mixed-generation consumers, peer dependencies, build constants, Tailwind, integrations, and CLI targeting |
| [Reactivity and async computations](references/reactivity-and-async.md) | Async work, task deprecations, store membership, `untrack()`, and computed notifications |
| [Router and server behavior](references/router-and-server.md) | Server errors, middleware responses, URL state, rewrites, mocks, origins, request events, redirects, and route-data caching |

## Breaking changes and deprecations

### Replace async `useComputed$` callbacks

Do not use an async callback with `useComputed$`. Signals first read after an
`await` are not tracked, and an initial promise restarts rendering. Move the
work to `useTask$` or `useResource$`; async computed callbacks stop working in
V2.

### Remove deprecated task eagerness

Do not add or preserve the `eagerness` option on `useTask$`. It is deprecated
from 1.13 and is removed in V2.

### Migrate away from built-in service-worker prefetching

From 1.14, use the bundle graph and generated `modulepreload` links for
prefetching. The built-in service-worker components are deprecated. For an
uncustomized worker, remove `service-worker.ts` but temporarily retain
`ServiceWorkerRegister` so deployed workers and caches are removed. Custom
worker logic prevents automatic unregistration; add the legacy integration
only when that custom behavior is still required.

```sh
qwik add service-worker
```

### Update preload configuration

Use stable `maxIdlePreloads` to limit concurrent idle preloads. Do not introduce
`preloadProbability`; it is deprecated from 1.16.1. Enable `debug` when preload
diagnostics are needed.

```ts
renderToStream(<Root />, {
  ...opts,
  preload: { debug: true, maxIdlePreloads: 5 },
});
```

### Align applications on Vite 7

Qwik core and Qwik City moved to Vite 7 in 1.16. Declare Vite directly in the
application because it is a peer dependency of Qwik, Qwik City, Qwik React,
and Qwik Labs.

### Republish Qwik libraries

Library builds from 1.9 no longer run the Qwik transform. Publish a fresh build
and extend the accepted Qwik range with `| ^2.0.0` when supporting V2 consumers.
The optimizer always processes library QRL files ending in `.qwik.js`,
`.qwik.mjs`, or `.qwik.cjs`; `fileFilter` cannot exclude them.

### Update asset and lint assumptions

Expect default build assets at `assets/hash-name.ext`, and update deployments
or cache rules that assume older locations. The `lint` option defaults to
`false`; enable it explicitly when linting is required.

## High-value feature guidance

### Configure experimental Vite behavior explicitly

Pass experiments through the `qwikVite()` `experimental` array:

```ts
qwikVite({ experimental: ['noSPA', 'valibot', 'preventNavigate'] });
```

- Use `noSPA` only for MPA-only applications that do not use `Link`.
- Use `valibot` with the experimental `valibot$` validator.
- Use `preventNavigate` with `usePreventNavigate`; it asynchronously blocks SPA
  navigation and falls back to browser dialogs for other unsaved-state exits.

### Use internal request rewrites

Call `RequestEvent.rewrite()` to serve another internal route while preserving
the browser-visible URL, and throw its result from the handler:

```ts
export const onRequest: RequestHandler = async ({ rewrite }) => {
  throw rewrite('/articles/42');
};
```

Multiple rewrite routes may share the same destination route.

### Use direct build constants

Import `isDev`, `isBrowser`, and `isServer` directly from `@builder.io/qwik`.
The older `@builder.io/qwik/build` entry point remains available.

```ts
import { isBrowser, isDev, isServer } from '@builder.io/qwik';
```

### Read values without tracking

Pass a signal or store directly to `untrack()`, or pass arguments through its
callback form:

```ts
const value = untrack(signal);
const result = untrack((a, b) => a + b, 1, 2);
```

Computed signals notify listeners only when the computed value itself changes,
not whenever a dependency changes to an equivalent result.

### Access raw store data deliberately

Use `unwrapStore()` when an underlying store value must cross into structured
cloning or IndexedDB:

```ts
import { unwrapStore } from '@builder.io/qwik';

const copy = structuredClone(unwrapStore(store));
```

### Customize MDX rendering

Pass a `components` map to imported MDX content. MDX JavaScript expressions can
read props, and a default-exported MDX layout component is honored.

```tsx
import { component$ } from '@builder.io/qwik';
import Content from './markdown.mdx';
import MyComponent from './my-component';

export default component$(() => <Content components={{ MyComponent }} />);
```

### Handle errors and transitions

Use `ErrorBoundary` for component error boundaries; corrected
`useErrorBoundary` behavior is available in 1.13. Listen for the
`qviewTransition` `CustomEvent` when a view transition starts.

## Server and deployment checks

### Treat server failures as throwable client errors

Server-function and route-loader errors share standardized behavior by 1.13.
Client calls throw for 4xx statuses and statuses above 500; status 499 is valid.
`@plugin` middleware can catch `server$` failures.

### Make cache behavior explicit

Serve content-hashed files under `build/` and `assets/` with
`Cache-Control: public, max-age=31536000, immutable` unless Rollup output names
were customized. Qwik City navigation honors `q-data.json` cache headers; its
default cache duration is one hour. Redirects do not inherit a parent layout's
`Cache-Control` and default to `no-store`.

### Verify loader and manifest delivery

SSR loads Qwikloader from a separate bundle from 1.15. Use
`qwikLoader: 'inline'` from 1.17 only for testing or unusual networks. The
preloader bundle graph is emitted as an asset, `q-manifest.json` contains
generated assets, and 1.19 filters `core.js` and `preloader.js` references from
both the manifest and graph. Run `qwik check-client` to verify client-bundle
freshness.

### Preserve request semantics

Handle the router's previous URL as `undefined` on first render. Treat request
events as readonly values rather than depending on runtime freezing. When
adapting Bun or Deno, use `getOrigin` in `QwikCityBunOptions` or
`QwikCityDenoOptions` to control request URL origins.

### Test route data and actions

Use `QwikCityMockProvider` to mock route loaders and actions. Middleware that
observes send-request events receives a `Response` even when the request
redirects.

## Additional authoring and tooling notes

- Checking `"prop" in store` creates a subscription and tracks later changes
  to that property's presence.
- Set `linkFetchPriority` on a prefetch strategy to control generated
  `modulepreload` links.
- The optimizer understands `import ... with`, replaces enums with numbers when
  possible, and delegates QRL grouping to Rollup so earlier Rollup/Vite plugins,
  including CSS-in-JS transforms, can process code first.
- Use the compiled i18n integration with `qwik add compiled-i18`.
- The Tailwind integration supports Tailwind CSS 4, while the CLI can retain
  Tailwind CSS 3 for projects that require it.
- Pass `projectDir` to `qwik add` when targeting a monorepo package or
  subproject rather than the repository root.
