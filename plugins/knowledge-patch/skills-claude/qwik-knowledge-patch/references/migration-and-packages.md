# Migration and Packages

## Publishing Qwik libraries

From 1.9, a Qwik library build no longer performs the Qwik transform. Publish a
new build rather than relying on output created under the older behavior. When
the library accepts both generations, extend its Qwik range with `| ^2.0.0`.

## Using V1 libraries in a V2 application

From 1.11, a V2 application can retain a V1 library by installing both core
generations:

```json
{
  "dependencies": {
    "@builder.io/qwik": "^1.11.0",
    "@qwik.dev/core": "^2.0.0"
  }
}
```

The V1 package supplies the retained library while the V2 package supplies the
application runtime generation.

## Vite peer dependencies

`vite` is a peer dependency of Qwik, Qwik City, Qwik React, and Qwik Labs to
avoid duplicate Vite imports. Add Vite directly to the application's
dependencies and align its version with the Qwik toolchain.

## Direct build-constant imports

Import the build constants from the main package:

```ts
import { isBrowser, isDev, isServer } from '@builder.io/qwik';
```

The older `@builder.io/qwik/build` entry point remains available.

## Tailwind integrations

The integration supports Tailwind CSS 4. The CLI also allows an existing
project to continue with Tailwind CSS 3.

## Compiled i18n

Scaffold the compiled-i18 integration directly:

```sh
qwik add compiled-i18
```

## Targeting integrations in a monorepo

Pass `projectDir` to `qwik add` when the integration belongs in a package or
subproject rather than at the repository root:

```sh
qwik add --projectDir=packages/my-package
```
