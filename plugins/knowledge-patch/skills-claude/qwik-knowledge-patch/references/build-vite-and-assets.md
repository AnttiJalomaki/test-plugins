# Build, Vite, and Assets

## Optimizer syntax and ordering

The optimizer understands `import ... with` and replaces enums with numbers
where possible. QRL grouping is delegated to Rollup, so other Rollup or Vite
plugins—including CSS-in-JS transforms—can process code before Qwik transforms
it.

## Default asset paths

Default build assets use `assets/hash-name.ext`. Update deployment rules, cache
configuration, and asset matching that assumed the former locations.

## Library QRL filtering

A custom `qwikVite()` `fileFilter` cannot exclude library QRL files named
`*.qwik.js`, `*.qwik.mjs`, or `*.qwik.cjs`; the optimizer always processes
them.

## Experimental Vite features

Configure experiments through an array:

```ts
qwikVite({ experimental: ['noSPA', 'valibot', 'preventNavigate'] });
```

- `noSPA` supports MPA-only applications that do not use `Link`.
- `valibot` enables the experimental `valibot$` validator.
- `preventNavigate` enables `usePreventNavigate`. It can asynchronously block
  SPA navigation and uses browser dialogs for other unsaved-state navigation.

## Module-preload priority

Set `linkFetchPriority` on a prefetch strategy to set the fetch priority of its
generated `modulepreload` links.

## Lint default

The `lint` option defaults to `false`. Enable it explicitly when a build must
run linting.

## Bundle preloading and service workers

From 1.14, Qwik prefetches through `modulepreload` links and a bundle graph that
contains dynamic imports and path-to-bundle mappings. The server includes
built-in manifest support; service workers are no longer the built-in prefetch
mechanism.

The built-in service-worker components are deprecated. For an uncustomized
worker, remove `service-worker.ts` but temporarily retain
`ServiceWorkerRegister` so existing workers and caches are removed from deployed
clients. Automatic unregistration does not occur when custom worker logic is
present. Install the integration only for a legacy application that still
needs a customizable worker:

```sh
qwik add service-worker
```

## SSR preload configuration

SSR preload options include `debug` and the stable `maxIdlePreloads`, which
limits concurrent idle preloads. `preloadProbability` is deprecated from
1.16.1.

```ts
renderToStream(<Root />, {
  ...opts,
  preload: { debug: true, maxIdlePreloads: 5 },
});
```

## Cache headers

Unless Rollup output naming is customized, serve content-hashed files below
`build/` and `assets/` with:

```http
Cache-Control: public, max-age=31536000, immutable
```

Qwik City's `q-data.json` requests follow their cache headers instead of being
forced fresh. Their default cache duration is one hour.

## Qwikloader delivery

From 1.15, SSR references Qwikloader as a separate bundle rather than embedding
it. From 1.17, tests and unusual network configurations can opt into embedding:

```ts
renderToStream(<Root />, { ...opts, qwikLoader: 'inline' });
```

## Manifest and preloader artifacts

The preloader bundle graph is emitted as an asset, and `q-manifest.json`
includes generated assets. By 1.19, references to `core.js` and `preloader.js`
are filtered from both the manifest and the bundle graph.

## Vite and client freshness

Qwik core and Qwik City moved to Vite 7 in 1.16. Align the application
toolchain with that major version. Use the CLI freshness check after producing
the client bundle:

```sh
qwik check-client
```
