# Framework and packages

## Choose the correct framework line

The direction announcement recorded in `remix-3-direction` separates two
paths:

- Remix v1/v2 continues through React Router v7 framework mode. That stable,
  long-term-supported path absorbed the earlier Remix bundler and server
  runtime.
- Remix 3 is a separate, non-React framework rebuilt around its own
  web-oriented component model. It is not the formerly planned RSC-first React
  successor and aims to have no critical dependencies, including React.
- At that announcement there was no Remix 3 preview.

React Router v7's then-previewed RSC support belongs to the incremental React
path. Its loaders and actions can return server components, and it supports
server-only routes, so an existing React application does not need Remix 3 to
adopt that architecture gradually.

## Understand the package direction

The early Remix 3 plan favors:

- Web APIs and JavaScript as the runtime contract.
- Standalone, composable packages with a zero-dependency goal.
- Runtime execution without required bundling, compilation, type generation,
  or other static analysis.
- `--import` loaders as the exception for TypeScript and JSX transforms.
- Packages that remain independently usable.

The early plan described re-exporting packages together from `remix`. Current
beta code instead removes the root export and uses domain-oriented subpaths.
The older one-to-one `@remix-run/*`-style aliases exist only for beta
migration and are planned for removal before stable. Treat the domain subpaths
as the operative import rule, while recognizing the early root-package plan as
historical intent rather than a current API.

## Use the beta as a preview

The cohesive beta preview in `remix-3-beta-preview` is intended for
experiments and prototypes, not production. It brings routing, sessions, auth,
forms, uploads, static and compiled assets, data, server rendering, and UI into
one preview.

Scaffold from the `next` release line:

```sh
npx remix@next new my-remix-app
```

## Keep runtime boundaries portable

Remix packages target Node.js, Bun, Deno, Cloudflare Workers, and other
JavaScript environments. At server boundaries, prefer browser-standard values
when available:

| Prefer | Avoid coupling to |
| --- | --- |
| Web Streams | Node streams |
| `Uint8Array` | `Buffer` |
| Web Crypto | `node:crypto` |
| `Blob` and `File` | Runtime-specific file APIs |

This contract is especially important for middleware, uploads, file storage,
and request handlers that may move between runtimes.

## Know the standalone runtime packages

- `assets` is a Fetch-based server that compiles browser JavaScript,
  TypeScript, and CSS on demand.
- `node-fetch-server` hosts Fetch API servers on Node.js.
- `node-tsx` runs Node.js entrypoints containing TypeScript and JSX syntax.

These pieces can be used independently rather than requiring a monolithic
compiler or server runtime.

## Select first-party adapters

The package set includes:

- Typed `data-table` adapters for MySQL, PostgreSQL, and SQLite.
- File storage expressed with JavaScript `File` objects, including an S3
  backend.
- Cookie-based session middleware and Memcache and Redis session-storage
  adapters.

For request handling, packages cover browser login, OAuth/OIDC, pluggable
authentication, `AsyncLocalStorage` request context, compression, tokenless
cross-origin protection, CORS, CSRF, streaming form-data uploads, logging,
method override, and static-file serving.

## Install a repository preview

With pnpm 9 or newer, install the built `preview/main` branch from either the
framework workspace or a standalone package workspace:

```sh
pnpm install "remix-run/remix#preview/main&path:packages/remix"
pnpm install "remix-run/remix#preview/main&path:packages/fetch-router"
```

Use this only when intentionally testing the bleeding-edge repository build;
the scaffold command above follows the packaged `next` line.
