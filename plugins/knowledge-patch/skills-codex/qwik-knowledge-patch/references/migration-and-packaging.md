# Migration and Packaging

Use this reference for library output, package compatibility, dependency
ownership, and public imports.

## Library builds and mixed-generation consumers

*Batch: `v1.8-1.13`*

From 1.9, Qwik library builds no longer perform the Qwik transform. Library
authors should publish a new build rather than assuming the old build output
remains suitable. When a library supports consumers on both generations,
extend its accepted Qwik range with `| ^2.0.0`.

A second-generation application can retain first-generation libraries by
installing both core generations:

```json
{
  "dependencies": {
    "@builder.io/qwik": "^1.11.0",
    "@qwik.dev/core": "^2.0.0"
  }
}
```

This is a deliberate dual-runtime compatibility setup, not permission to
remove the first-generation dependency while retained libraries still import
it. Test library component rendering and server output after changing either
range.

## Vite dependency placement

*Batch: `v1.8-1.13`*

Vite is a peer dependency of Qwik, Qwik City, Qwik React, and Qwik Labs.
Applications must depend on Vite directly. This prevents duplicate Vite
imports and gives the application control of the resolved toolchain version.

Check `package.json` and the lockfile:

- the application declares Vite directly;
- workspace packages do not accidentally install private Vite copies; and
- plugin resolution reaches the same Vite instance used to run the build.

Qwik core and Qwik City later moved to Vite 7; see
[Build and deployment](build-and-deployment.md#vite-7).

## Direct build-constant exports

*Batch: `v1.8-1.13`*

Import `isDev`, `isBrowser`, and `isServer` directly from
`@builder.io/qwik`:

```ts
import { isBrowser, isDev, isServer } from '@builder.io/qwik';
```

The older `@builder.io/qwik/build` entry point remains available. New code can
use the root exports without forcing an immediate rewrite of existing imports.

## Package and build verification

After changing library packaging:

1. build the library from a clean checkout;
2. inspect published files rather than only source output;
3. install the packed artifact into a representative application;
4. verify the application owns the Vite dependency;
5. run both client and SSR builds; and
6. check that QRL-bearing library files are processed by the application
   build.

The QRL filter rule is documented in
[Build and deployment](build-and-deployment.md#library-qrl-file-filtering).
