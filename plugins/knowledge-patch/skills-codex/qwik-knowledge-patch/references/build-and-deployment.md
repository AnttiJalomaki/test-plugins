# Build and Deployment

Use this reference for optimizer behavior, Vite configuration, asset output,
preloading, manifests, service workers, CLI integrations, and cache policy.

## Optimizer syntax and transform ordering

*Batch: `v1.8-1.13`*

The optimizer understands import attributes written with `with`:

```ts
import data from './data.json' with { type: 'json' };
```

It also replaces enums with numbers when possible. Do not rely on an emitted
runtime enum object when the optimizer can erase it.

QRL grouping is delegated to Rollup. This lets other Rollup and Vite plugins,
including CSS-in-JS transforms, process code before Qwik transforms it. When a
plugin-order bug appears:

1. inspect the effective Vite plugin order;
2. confirm the upstream plugin returns the expected transformed module;
3. inspect the module received by the Qwik transform; and
4. avoid compensating for stale assumptions about Qwik grouping internally.

## Asset paths and cache headers

*Batches: `v1.8-1.13` and `v1.14-1.19`*

Default build assets use this shape:

```text
assets/hash-name.ext
```

Update deployment upload rules, CDN routing, CSP asset enumeration, and cache
invalidation that assumed an older location.

Content-hashed files under `build/` and `assets/` should normally use:

```http
Cache-Control: public, max-age=31536000, immutable
```

Do not apply that policy blindly when Rollup output naming was customized and
the names are not content hashed.

Qwik City navigation no longer forces a fresh `q-data.json` download.
Navigation follows the response's cache headers, and the default cache
duration is one hour. Set route-data cache headers deliberately when data is
more volatile or user-specific.

## Library QRL file filtering

*Batch: `v1.8-1.13`*

A custom `qwikVite()` `fileFilter` cannot exclude these library QRL files:

- `*.qwik.js`
- `*.qwik.mjs`
- `*.qwik.cjs`

They are always processed. Keep the filter focused on other file classes and
fix incompatible published QRL output at its source instead of attempting to
skip the transform.

## Experimental feature gating

*Batch: `v1.8-1.13`*

`qwikVite()` accepts an `experimental` array:

```ts
qwikVite({
  experimental: ['noSPA', 'valibot', 'preventNavigate'],
});
```

The features have distinct scopes:

- `noSPA` targets MPA-only applications that do not use `Link`.
- `valibot` enables the experimental `valibot$` validator.
- `preventNavigate` enables `usePreventNavigate`. It can asynchronously block
  SPA navigation and falls back to browser dialogs for other unsaved-state
  navigation.

Gate only the features the application uses. Code that calls an experimental
API must keep the matching feature name in the effective Vite configuration.

## Module-preload fetch priority

*Batch: `v1.8-1.13`*

Prefetch strategies can set `linkFetchPriority` for generated
`modulepreload` links. Use it when the browser needs an explicit priority hint
for modules selected by the prefetch strategy.

Inspect rendered link elements to verify the option reached SSR output, and
measure the result under representative network conditions before assigning
high priority broadly.

## Monorepo-aware `qwik add`

*Batch: `v1.8-1.13`*

Pass `projectDir` when an integration should target a package or subproject
instead of the repository root:

```sh
qwik add --projectDir=packages/my-package
```

## Tailwind integrations

*Batch: `v1.8-1.13`*

The Qwik integration supports Tailwind CSS 4. The CLI also permits projects to
continue using Tailwind CSS 3.

Choose the Tailwind major that matches the project's existing configuration
and plugin ecosystem. Do not treat running the Qwik integration as an
automatic Tailwind-major migration.

## Lint default

*Batch: `v1.8-1.13`*

The Qwik `lint` option defaults to `false`. Enable it explicitly when the
build or integration is expected to lint:

```ts
qwikVite({
  lint: true,
});
```

Keep a separate CI lint command when linting should remain mandatory even for
build invocations that disable plugin linting.

## Automatic preloading and service-worker migration

*Batch: `v1.14-1.19`*

Qwik prefetching now uses `modulepreload` links and a bundle graph instead of
the built-in service workers. The graph includes dynamic imports and
path-to-bundle mappings, and the server has built-in manifest support.

The built-in service-worker components are deprecated. For an uncustomized
worker:

1. remove `service-worker.ts`;
2. temporarily retain `ServiceWorkerRegister`;
3. deploy the transition;
4. verify previously deployed workers and caches are removed; and
5. remove the temporary registration only after the deployed population has
   had an opportunity to run the cleanup.

Custom service-worker logic prevents automatic unregistration. Keep or add the
integration when a legacy application still needs a customizable worker:

```sh
qwik add service-worker
```

Audit the worker before applying the uncustomized-worker removal sequence.

## SSR preload configuration

*Batch: `v1.14-1.19`*

SSR preload options include `debug` and `maxIdlePreloads`:

```ts
renderToStream(<Root />, {
  ...opts,
  preload: {
    debug: true,
    maxIdlePreloads: 5,
  },
});
```

`maxIdlePreloads` is the stable limit on concurrent idle preloads. Use
`debug` while inspecting preload decisions, then choose a limit based on
network and page behavior.

`preloadProbability` is deprecated from 1.16.1. Remove it rather than
assuming it remains part of the supported preload policy.

## Qwikloader delivery

*Batch: `v1.14-1.19`*

SSR normally loads the Qwikloader from a separate bundle instead of embedding
it in the document. This affects CSP rules, asset delivery, and tests that
expected inline loader text.

Testing and unusual network setups can opt back into embedding:

```ts
renderToStream(<Root />, {
  ...opts,
  qwikLoader: 'inline',
});
```

Prefer the default external bundle in ordinary deployments. When using inline
delivery, confirm the document's script policy permits it.

## Manifest and preloader artifacts

*Batch: `v1.14-1.19`*

The preloader bundle graph is emitted as an asset. `q-manifest.json` includes
generated assets, so consumers should tolerate and process asset entries
rather than assuming it lists only application chunks.

By 1.19, `core.js` and `preloader.js` references are filtered from both the
manifest and bundle graph. Deployment tools must not require those entries to
exist.

When consuming build metadata:

- locate the emitted graph through build output rather than a hard-coded
  source path;
- accept generated assets in `q-manifest.json`;
- do not synthesize missing `core.js` or `preloader.js` entries; and
- test custom Rollup naming together with asset upload and cache rules.

## Vite 7

*Batch: `v1.14-1.19`*

Qwik core and Qwik City moved to Vite 7 in 1.16. Their application toolchain
must use that major version.

Declare Vite directly in the application, update Vite plugins to compatible
releases, and inspect the lockfile for duplicate older majors. Exercise both
development and production builds because plugin compatibility can differ
between them.

For dependency ownership details, see
[Migration and packaging](migration-and-packaging.md#vite-dependency-placement).

## Client-bundle freshness check

*Batch: `v1.14-1.19`*

Use the CLI's `check-client` command to verify that the client bundle is
fresh:

```sh
qwik check-client
```

Include this check where stale client output would otherwise be deployed
alongside newer server output. A failure should trigger a client rebuild, not
manual timestamp adjustment.

## Compiled i18n integration

*Batch: `v1.14-1.19`*

Qwik can scaffold compiled internationalization support directly:

```sh
qwik add compiled-i18
```

Inspect the generated Vite and application changes before combining them with
an existing localization pipeline. In a workspace, add `projectDir` when the
integration belongs to a subproject.
