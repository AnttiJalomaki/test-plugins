# Runtime, CLI, and testing

Use this reference for portable server boundaries, on-demand asset compilation,
Node integration, command-line requirements, TypeScript/JSX execution, and test
timeouts.

## Portable server contract

Remix 3 packages are intended to run across JavaScript environments including:

- Node.js;
- Bun;
- Deno;
- Cloudflare Workers; and
- other environments that implement the required Web APIs.

When the environment provides them, server-side code uses browser-standard
data types and APIs:

| Prefer | Avoid at portable boundaries |
| --- | --- |
| Web Streams | Node streams |
| `Uint8Array` | `Buffer` |
| Web Crypto | `node:crypto` |
| `Blob` and `File` | Runtime-specific file APIs |

This is a package-boundary rule, not a claim that platform-native APIs never
exist. Keep runtime-specific integration behind an adapter when portable Remix
packages exchange these standard values.

## Standalone runtime packages

The `assets` package is a Fetch-based server that compiles browser JavaScript,
TypeScript, and CSS on demand.

`node-fetch-server` hosts a Fetch API server on Node.js. Its `trustProxy`
security implications are covered in
[data-http-and-security.md](data-http-and-security.md).

`node-tsx` runs Node.js with TypeScript and JSX syntax support. These packages
are independently usable; installing Remix 3 does not imply a mandatory static
build or compiler stage.

## CLI package and Node requirement

The package exposes both:

- the `remix` binary; and
- the `remix/cli` programmatic entrypoint.

CLI metadata requires Node.js 24.3.0 or later. Check the executing Node binary,
not merely the version used to install dependencies, when CLI startup fails.

## Node TypeScript and JSX loader

Run a TypeScript/JSX entrypoint through the package loader with:

```sh
node --import remix/node-tsx app.tsx
```

The loader retains Node module resolution and transforms TypeScript syntax that
requires emitted JavaScript, including:

- enums;
- runtime namespaces; and
- parameter properties.

This differs from type-stripping approaches that cannot execute those
constructs.

## Test command and cancellation

The `remix-test` binary no longer exists. Run:

```sh
remix test
```

The command uses the internal `node-tsx` loader. Tests and hooks accept an
options object containing `{ timeout, signal }`. When a test times out,
`t.signal` aborts.

Long-running helpers should observe the signal and stop work on cancellation;
otherwise a timed-out test may leave background operations running even though
the test runner has moved on.

## `node-serve` availability

An early beta exposed `remix/node-serve`, but beta 2 removed both
`remix/node-serve` and `@remix-run/node-serve`. The native transport dependency
could not be delivered in npm-compatible packaging. The package remains absent
through beta 5, with restoration deferred to a later beta.

Do not diagnose the missing import as an installation corruption. Use an
available hosting option, such as the Fetch-to-Node integration appropriate to
the project, until the native server package is restored.

## Repository preview installs

With pnpm 9 or newer, the built `preview/main` branch can be selected at a
workspace path:

```sh
pnpm install "remix-run/remix#preview/main&path:packages/remix"
pnpm install "remix-run/remix#preview/main&path:packages/fetch-router"
```

The first command selects the full framework workspace; the second selects the
Fetch router workspace. Pin the intended repository reference in reproducible
experiments because the preview branch is a bleeding-edge source.

## Runtime debugging checklist

1. Check Node.js 24.3.0 or later before debugging the CLI.
2. Confirm whether the entrypoint uses `--import remix/node-tsx` when it contains
   TypeScript or JSX.
3. Treat missing `node-serve` as expected in the current betas.
4. Keep Fetch request/response and browser-standard binary types at portable
   package boundaries.
5. Observe test and hook abort signals in long-running asynchronous helpers.
6. Inspect `trustProxy` before accepting forwarded URLs or client addresses.
