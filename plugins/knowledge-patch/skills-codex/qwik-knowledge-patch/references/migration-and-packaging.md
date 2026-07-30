# Migration and packaging

## Package and type changes

Qwik City is named Qwik Router in V2 and is published as `@qwik.dev/router`.
Core and Router publish ESM only; CJS and UMD builds are not available.

The V2 type surface removes the old HTML-specific types in favor of `PropsOf`
and trims other TypeScript exports. Core exposes `HTMLElementAttrs`, `SVGProps`,
and `SVG` where explicit element and SVG typing is needed.

Build constants can be imported directly from core:

```ts
import { isBrowser, isDev, isServer } from '@builder.io/qwik';
```

The V1 `@builder.io/qwik/build` entry point remains available. After migration,
use the corresponding V2 core package path.

## Automated V2 migration

Run the migration before doing manual cleanup:

```sh
pnpm qwik migrate-v2
```

It updates package names, renamed identifiers, Qwik and Vite configuration, and
dependencies. Check these less-obvious mappings:

- `@builder.io/qwik-react` becomes `@qwik.dev/react`.
- `@qwik-city-plan` becomes `@qwik-router-config`.
- The JSX runtime becomes `@qwik.dev/core/jsx-runtime`.

Finish the application configuration explicitly:

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

All `@qwik.dev/*` packages belong in `devDependencies`. V2 supports Vite 8 and
Rolldown, so migrated tooling can use those when the rest of the project is
compatible.

## V1 libraries and V2 consumers

From Qwik 1.9, library builds stopped running the Qwik transform. Library
authors should publish a new build and extend the accepted Qwik range with
`| ^2.0.0`.

A staged V2 consumer can deliberately retain V1 libraries by installing both
generations:

```json
{
  "dependencies": {
    "@builder.io/qwik": "^1.11.0",
    "@qwik.dev/core": "^2.0.0"
  }
}
```

This is a compatibility bridge, not a substitute for updating the library.
The V2 qwikloader does not process V1 containers, so any page that retains
those containers must also load the V1 qwikloader.

## Third-party library overrides

Libraries that still name V1 packages can otherwise install a second Qwik
runtime. Override or resolve both package names to V2:

```json
{
  "overrides": {
    "@builder.io/qwik": "npm:@qwik.dev/core@^2",
    "@builder.io/qwik-city": "npm:@qwik.dev/router@^2"
  }
}
```

Also bundle the library into the server output and keep it out of Vite
dependency optimization:

```ts
export default defineConfig({
  ssr: { noExternal: ['some-qwik-library'] },
  optimizeDeps: { exclude: ['some-qwik-library'] },
});
```

Missing either setting can produce `Code(Q30)` and external-dependency
warnings.

## Replacing Qwik Labs

The `qwik-labs` package is removed. Insights now uses core's `insights`
experiment and core entry points:

```ts
import { qwikInsights } from '@qwik.dev/core/insights/vite';

qwikVite({ experimental: ['insights'] });
```

```tsx
import { Insights } from '@qwik.dev/core/insights';

<Insights publicApiKey="..." postUrl="..." />
```

Typed routes are built into Qwik Router through `qwikTypes()`; do not look for
a replacement package for the removed Labs implementation.

## Migrating the router root

When removing V1's `<QwikCityProvider>`, a non-reactive root may call
`useQwikRouter()` directly while rendering `RouterOutlet`. The hook runs only
once during SSR. If the root itself reads signals and must update reactively,
render `<QwikRouterProvider>` instead.

## Serialized state tooling

V2 no longer emits one `<script type="qwik/json">` element. It emits separate
`<script type="qwik/vnode">` and `<script type="qwik/state">` elements at the
end of the document. Update HTML transforms, security policies, and state
extractors to recognize both tag types and their placement.
