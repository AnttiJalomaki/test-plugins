# Migration and Packages

Use this reference for V1-to-V2 conversion, dependency ownership, public type
changes, and third-party library compatibility.

Source coverage includes `v1.8-1.13`, `v1.14-1.19`,
`v2-alpha-beta-1-9`, `v2-beta-10-19`, `v2-beta-20-29`,
`v2-beta-30-38`, and `migration-v1-v2`.

## Run the migration first

Run the migration CLI before manual cleanup:

```sh
pnpm qwik migrate-v2
```

It updates package names, renamed identifiers, Qwik and Vite configuration,
and dependencies. Important mappings include:

| V1 name | V2 name |
| --- | --- |
| `@builder.io/qwik` | `@qwik.dev/core` |
| `@builder.io/qwik-city` | `@qwik.dev/router` |
| `@builder.io/qwik-react` | `@qwik.dev/react` |
| `@qwik-city-plan` | `@qwik-router-config` |
| V1 JSX runtime | `@qwik.dev/core/jsx-runtime` |

Qwik City is named Qwik Router in V2, and its package is
`@qwik.dev/router`.

## Project configuration

V2 supports Vite 8 and Rolldown. A migrated project must:

- use Bundler module resolution;
- use the V2 JSX import source;
- declare `"type": "module"` in `package.json`; and
- place every `@qwik.dev/*` package in `devDependencies`.

```json
{
  "type": "module",
  "devDependencies": {
    "@qwik.dev/core": "^2",
    "@qwik.dev/router": "^2"
  }
}
```

```json
{
  "compilerOptions": {
    "moduleResolution": "Bundler",
    "jsxImportSource": "@qwik.dev/core"
  }
}
```

Core and Router are ESM-only; CJS and UMD distributions are no longer
published. Consumers, tests, and build tools must load ESM.

## Type-surface changes

V2 removes the broad set of HTML-specific types in favor of `PropsOf` and
reduces the general TypeScript export surface. Later prereleases make
`HTMLElementAttrs`, `SVGProps`, and `SVG` available from Core.

Route-loader signal typing and its ESLint rule were expanded, so use current V2
types and lint rules when broader signal usage is reported incorrectly.

## Vite dependency ownership

Vite is a peer dependency of Qwik, Qwik City/Router, Qwik React, and Qwik Labs,
which avoids duplicate Vite imports. Applications must install Vite directly.
Qwik Core and Qwik City moved to Vite 7 in 1.16; align a V1 application's
toolchain to that major when upgrading within V1.

The V2 Environment API integration has a minimum supported Vite version of
6.0.0 for multi-environment `vite build --app` support. This does not conflict
with migrated projects supporting later Vite versions.

## Publishing Qwik libraries

From Qwik 1.9, library builds no longer run the Qwik transform. Library authors
should publish a newly built package and extend the accepted Qwik range with
`| ^2.0.0`.

A V2 consumer can retain a V1 library by installing both generations:

```json
{
  "dependencies": {
    "@builder.io/qwik": "^1.11.0",
    "@qwik.dev/core": "^2.0.0"
  }
}
```

The V2 qwikloader cannot run V1 containers, so a document that actually
retains V1 containers must also load the V1 qwikloader.

## Third-party V1 dependencies

If a dependency still names the V1 packages, prevent it from installing a
second runtime with package-manager overrides or resolutions:

```json
{
  "overrides": {
    "@builder.io/qwik": "npm:@qwik.dev/core@^2",
    "@builder.io/qwik-city": "npm:@qwik.dev/router@^2"
  }
}
```

Bundle the dependency into the server output and keep it out of Vite dependency
optimization:

```ts
export default defineConfig({
  ssr: { noExternal: ['some-qwik-library'] },
  optimizeDeps: { exclude: ['some-qwik-library'] },
});
```

Without all three measures, the project can load duplicate runtimes or produce
`Code(Q30)` and external-dependency warnings.

Qwik no longer scans every module at startup to discover which Qwik modules
must be included in server output. Its faster build-time check reports when
`ssr.noExternal` needs adjustment; treat that diagnostic as a dependency
configuration error.

## Labs features after package removal

The `qwik-labs` package is removed.

Insights is a Core experiment. Import its Vite plugin and component from Core:

```ts
import { qwikInsights } from '@qwik.dev/core/insights/vite';

qwikVite({ experimental: ['insights'] });
```

```tsx
import { Insights } from '@qwik.dev/core/insights';

<Insights publicApiKey="..." postUrl="..." />
```

Typed routes are built into Qwik Router through `qwikTypes()`; there is no
replacement Labs package to install.

## Optimizer and adapter package cleanup

Optimizer bindings live in `@qwik.dev/optimizer`.
`@qwik.dev/core/optimizer` re-exports the runtime for compatibility, while the
Vite plugin remains bundled with Core.

Development QRL segment mapping is handled by Core, so remove a separate
`symbolMapper` function. The unused `qwikVite()` `srcInput` option is
deprecated.

Server chunks now live in their own `build/` subdirectory. Forked Router
adapters must remove imports of `@qwik-router-not-found-paths` and
`@qwik-router-static-paths`; that metadata is embedded directly.

## Migration searches

Search a migrated tree for:

```text
@builder.io/qwik
@builder.io/qwik-city
@builder.io/qwik-react
@qwik-city-plan
qwik-labs
@qwik.dev/core/optimizer
symbolMapper
srcInput
@qwik-router-not-found-paths
@qwik-router-static-paths
```

Retain an old package name only when it is an intentional compatibility
override or a deliberately co-installed V1 runtime.
