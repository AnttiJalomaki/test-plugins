---
name: qwik-knowledge-patch
description: Qwik
version: 2.0.0-beta.38
license: MIT
metadata:
  author: Nevaberry
---


# Qwik Compatibility Guide

Use this skill when upgrading, building, debugging, or reviewing modern Qwik
Core and Qwik Router applications. Start with the quick references below, then
open the topic file that matches the work.

## Reference map

| Reference | Topics |
| --- | --- |
| [migration-and-packages.md](references/migration-and-packages.md) | V1-to-V2 migration, package renames, ESM, libraries, type and config changes |
| [build-vite-and-assets.md](references/build-vite-and-assets.md) | Optimizer, Vite, manifests, preloading, assets, HMR, SSG, integrations |
| [reactivity-and-async.md](references/reactivity-and-async.md) | Signals, stores, tasks, async computations, resources, Suspense |
| [router-and-server.md](references/router-and-server.md) | Router context, loaders, actions, middleware, errors, rewrites, caching, SSR |
| [jsx-events-and-serialization.md](references/jsx-events-and-serialization.md) | JSX attributes, event naming, directives, MDX, serializers, runtime loader |

## Upgrade first: package and project changes

Run the automated migration before making manual edits:

```sh
pnpm qwik migrate-v2
```

Apply these package mappings:

| Earlier package or import | V2 replacement |
| --- | --- |
| `@builder.io/qwik` | `@qwik.dev/core` |
| `@builder.io/qwik-city` | `@qwik.dev/router` |
| `@builder.io/qwik-react` | `@qwik.dev/react` |
| `@qwik-city-plan` | `@qwik-router-config` |
| old JSX runtime | `@qwik.dev/core/jsx-runtime` |
| optimizer bindings | `@qwik.dev/optimizer` |

For a migrated project:

- Declare `"type": "module"`; Core and Router are ESM-only.
- Use TypeScript `"moduleResolution": "Bundler"`.
- Set `"jsxImportSource": "@qwik.dev/core"`.
- Put all `@qwik.dev/*` packages in `devDependencies`.
- Depend on Vite directly; it is a peer dependency of the Qwik packages.
- Replace broad HTML-specific types with `PropsOf` where possible.
- Remove `qwik-labs`; Insights comes from Core and typed routes from Router.

Third-party libraries that still name V1 packages need package-manager
overrides to the V2 packages. Also add those libraries to `ssr.noExternal` and
`optimizeDeps.exclude`; otherwise a second runtime, `Code(Q30)`, or
external-dependency warnings can result.

See [migration-and-packages.md](references/migration-and-packages.md) for
library-author compatibility, adapter cleanup, and the complete migration
details.

## Choose the current async API

The preferred API for an asynchronous reactive computation is again
`useComputed$()`. `useAsync$()`, `createAsync$()`, and `AsyncSignal` are
deprecated.

```tsx
const item = useComputed$(async ({ abortSignal, track }) => {
  const id = track(() => idSignal.value);
  const response = await fetch(`/api/items/${id}`, {
    signal: abortSignal,
  });
  return response.json() as Promise<Item>;
});

if (item.pending) return <p>Loading...</p>;
if (item.error) return <p>{item.error.message}</p>;
return <p>{item.value.name}</p>;
```

Remember:

- Reads before the first `await` are auto-tracked; call context `track()` for
  dependencies read afterward.
- `.value` is the resolved `T`, not `Promise<T>`. It throws while unresolved
  and rethrows a computation failure.
- `.pending` reports unresolved state and `.error` exposes failures.
- Context supplies `abortSignal` and `previous`.
- Options include `initial`, `clientOnly`, `expires` with `poll`, and
  `concurrency`; `0` means unlimited parallel work.
- `useResource$()` and `<Resource />` are deprecated.

When maintaining intermediate `useAsync$()` code, do not expect `.promise()` to
return the calculated value: it returns `Promise<void>`. Read `.value` and
`.error` instead. The option once named `interval` is now `expires`; `poll`
controls reruns after expiry.

For scheduling, cancellation, stale-result protection, and task cleanup, open
[reactivity-and-async.md](references/reactivity-and-async.md).

## Router migration and navigation

Qwik City is Qwik Router in V2. A non-reactive root can remove
`<QwikCityProvider>`, call `useQwikRouter()`, and render `RouterOutlet`.
Because that hook runs only once during SSR, a root that reads signals must use
`<QwikRouterProvider>` instead.

SPA view transitions are opt-in:

```tsx
<QwikRouterProvider viewTransition={true}>
  <RouterOutlet />
</QwikRouterProvider>
```

Navigation and request control use these current rules:

- `Link.prefetchBundle` is renamed `prefetchBundles`.
- `usePreventNavigate$()` and `request.rewrite()` are stable.
- Returning `ev.redirect()`, `ev.error()`, or `ev.rewrite()` has the same
  control-flow effect as throwing it.
- The first router previous URL is `undefined`.
- An invalid absolute rewrite returns `400`.
- Redirects default to `Cache-Control: no-store`.
- `RequestEvent.internalRequest` distinguishes framework-internal JSON
  requests.

## Route-loader checklist

Route loaders now behave as async signals:

- A thrown failure populates `.error`; reading `.value` rethrows it.
- `fail()` remains a distinct result whose value is `{ failed }`.
- Options include `expires`, `poll`, and `allowStale`.
- A loader cannot read action state. Read it in the component or put relevant
  state in the URL.
- With strict loaders enabled by default, only query keys declared by `search`
  are received and trigger reruns. No `search` means no query parameters and
  no query-driven rerun.
- `blockSSR` defaults to `true`; experimental background loading uses
  `blockSSR: false`.

SPA loader transport uses manifest-versioned
`q-loader-${hash}.${manifestHash}.json` files. Loader expiry defaults to two
minutes; `expiry: 0` disables browser caching and emits static loader data for
SSG. Keep expiry short and use ETag validation for user-specific data.

See [router-and-server.md](references/router-and-server.md) for `routeConfig`,
ETags, cache keys, head merging, error routes, and middleware behavior.

## Build and deployment checklist

- Built asset placement follows Rollup `output.assetFileNames`; do not rely on
  Qwik handling `build.assetsDir`.
- If the client manifest is not in its normal location, set
  `ssr.manifestInputPath`.
- Use `getClientManifest()` rather than importing
  `@qwik-client-manifest`.
- Client builds reject server-only imports.
- Code that calls Vite programmatically must use
  `createBuilder().buildApp()` when SSG is expected.
- Router adapters still require a separate `build.server` even when other
  environments build together with `vite build --app`.
- Serve content-hashed files under `build/` and `assets/` with a one-year,
  immutable cache policy unless output naming is customized.
- Use `qwikLoader: 'inline'` only for tests or unusual network delivery; the
  normal loader is a separate bundle.

The old built-in service-worker prefetch components are deprecated. For an
uncustomized worker, remove `service-worker.ts` but temporarily retain
`ServiceWorkerRegister` so deployed workers and caches can be removed. Keep or
add the service-worker integration only for custom worker logic.

Open [build-vite-and-assets.md](references/build-vite-and-assets.md) before
changing optimizer ordering, preload behavior, output directories, HMR, or
prerendering.

## Events, serialization, and rendering traps

Custom JSX event spelling is significant:

| JSX handler | Event matched |
| --- | --- |
| `onCustomEvent$` | `customevent` |
| `on-CustomEvent$` | `CustomEvent` |
| `onCustom-Event$` | `custom-event` |

Use kebab-case event segments in `preventdefault:event` and
`stoppropagation:event`. Handlers also recognize `passive:eventname` and
`capture:eventname` markers. Suppress a targeted optimizer diagnostic with
`@qwik-disable-next-line` only where necessary.

V2 serialized state appears as separate `qwik/vnode` and `qwik/state` scripts
at the end of the document, not as a `qwik/json` script. Custom serializers
use `useSerializer$()` or `createSerializer$()`, and native serialization
supports Temporal values.

See [jsx-events-and-serialization.md](references/jsx-events-and-serialization.md)
for promise-valued attributes, spread bindings, MDX customization, scoped
styles, Qwikloader lifecycle, and serializer symbols.

## Verification habits

When reviewing a migration:

1. Check `package.json`, TypeScript JSX settings, Vite ownership, and ESM mode.
2. Search for old package names, Labs imports, optimizer imports, provider
   components, and serialized-state selectors.
3. Search for `useAsync$`, `useResource$`, `.resolve()`, `.promise()`,
   `prefetchBundle`, `build.assetsDir`, and removed preload options.
4. Exercise SSR, SPA navigation, error routes, loader query changes, and a
   second SSG run.
5. Test custom event casing, server-only import rejection, and third-party
   libraries in both client and server builds.
