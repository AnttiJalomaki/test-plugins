---
name: remix-knowledge-patch
description: Remix
version: 3.0.0-beta.5
license: MIT
metadata:
  author: Nevaberry
---


# Remix 3

Use this skill when designing, migrating, reviewing, or debugging Remix 3
applications and packages. It captures the standalone web-platform framework
line, whose APIs differ substantially from Remix v1/v2 and React Router
framework mode.

## Start here

1. Inspect `package.json`, imports, generated app structure, and the runtime
   entrypoint before proposing changes.
2. Distinguish Remix 3 code from the React-based Remix v1/v2 lineage. Do not
   apply React Router framework conventions to the new component runtime.
3. Treat beta-era aliases and removed exports as migration clues, not as the
   preferred API.
4. Prefer Web API values at runtime boundaries and domain-oriented `remix/*`
   imports.
5. Preserve explicit middleware continuation, typed route context, and abort
   handling in asynchronous UI work.
6. Open the relevant reference below before changing code.

## Reference index

| Reference | Topics |
| --- | --- |
| [Framework and packages](references/framework-and-packages.md) | Framework lineage, beta status, runtime contract, package layout, assets, adapters, repository installs |
| [Routing and middleware](references/routing-and-middleware.md) | Typed route maps, method routes, mounts, app actions, context transforms, route patterns, dispatch errors |
| [UI and frames](references/ui-and-frames.md) | Handles, updates, event mixins, hydration, frames, UI import migrations, popovers, preserved DOM, CSS layers |
| [HTTP, security, and storage](references/http-security-and-storage.md) | Proxy trust, cookies, multipart parsing, auth, origin protection, CSRF, uploads, file and session storage |
| [Data, CLI, and testing](references/data-cli-and-testing.md) | Data-table APIs, SQL changes, CLI requirements, Node loader, tests, removed server transport |

## Breaking changes first

### Import from domain subpaths

The `remix` root export is absent. Import from focused entrypoints such as:

```ts
import { createRouter } from "remix/router";
import { route } from "remix/routes";
import { requireAuth } from "remix/middleware/auth";
```

Use `remix/ui` for the component runtime and its JSX/server subpaths. Treat the
old one-to-one package aliases as temporary migration aids.

### Continue middleware explicitly

Every router middleware function must return a `Response` or return `next()`.
Falling through with `undefined` throws:

```ts
function loadUser(): Middleware {
  return (context, next) => {
    context.set(CurrentUser, user);
    return next();
  };
}
```

Apply the same rule to global, controller, action, and method-helper
middleware.

### Use app actions, not app controllers

Generated applications and route-inspection commands expect action files under
`app/actions`. Put root actions in `app/actions/controller.tsx`, and map nested
route maps explicitly from `app/router.ts`. Controller middleware affects only
that controller's direct actions.

### Update route-pattern imports

Keep parsing and serialization at `remix/route-pattern`. Import URL generation,
matching, joining, and ranking from `/href`, `/match`, `/join`, and
`/specificity`. Matching is always most-specific-first; explicitly sort
`matchAll()` results when another order is required.

### Update UI imports and removed helpers

Replace `remix/component*` and `remix/components/*` imports with `remix/ui`,
component subpaths such as `remix/ui/menu`, or their `/primitives` variants.
Do not restore removed `glyph`, `scroll-lock`, `separator`, or `theme`
subpaths, or removed root helper re-exports.

### Update data-table code

Import `SqlStatement`, `sql`, and `rawSql` from `remix/data-table`.
`Database` is constructible, and `QueryBuilder` has been replaced by
`Query`/`query`. Terminal operations on an unbound query produce `Query`
objects; `db.exec(...)` accepts either raw SQL or a `Query`.

### Account for CLI and server changes

Use the `remix` binary or `remix/cli`. The separate `remix-test` binary is
absent. TypeScript/JSX Node entrypoints use:

```sh
node --import remix/node-tsx app.tsx
```

Do not import `remix/node-serve`; the native transport package is not present
in the current beta packaging.

## High-value runtime patterns

### Build typed route maps separately from handlers

Define the named route tree with `route()`, then map handlers with
`router.map()`. String leaves accept any method until registration;
`{ method, pattern }` leaves carry a fixed method. Every leaf has a typed
`href()` for links and form actions.

Use `form()` for paired `GET`/`POST` routes and `resources()` or `resource()`
for conventional RESTful maps. See the routing reference for generated names,
filters, and parameter controls.

### Mount reusable route installers

`router.mount(pattern, installer)` registers a feature's routes beneath a
parent pattern. The installer receives a `RouteBuilder`; the parent router
retains dispatch, matching, and its default handler. Add an installer catch-all
only when the mounted feature itself should own unmatched paths.

### Carry context types through middleware

Built-in middleware can add direct properties such as `auth`, `formData`,
`logger`, `render`, and `session`. Configure the application context contract
once, then derive transformed context types with `RouterContext`,
`MiddlewareContext`, or `ContextWithEntry`.

Inline middleware arrays infer context directly. Reach for
`createMiddleware()` when a chain crosses an inference boundary, such as an
exported value, factory result, or standalone type derivation.

### Use the component handle lifecycle

A component receives a `Handle`, initializes mutable state in its function,
and returns a render callback. Compose element behavior through `mix`.
After changing local state, call `handle.update()`.

Asynchronous `on()` mixins receive an abort signal. After every awaited
operation, return without updating if `signal.aborted` is true.

### Use frames for asynchronous HTML UI

`Frame` can nest with hydrated components. Mutation revalidation returns HTML
that is morphed into the existing DOM. Use `navigate`, `link`, `run`, and the
frame handles from `remix/ui`; build server-rendered sources with `frameSrc`
and `topFrameSrc` from `remix/ui/server`.

Mark the smallest client-owned widget with `rmx-preserve-dom` when later frame
updates must retain its live attributes and children.

## HTTP and security checks

### Trust proxy headers only behind a trusted proxy

Set `trustProxy` on `remix/node-fetch-server` only when the deployment boundary
can safely supply `Forwarded` or `X-Forwarded-*`. It changes reconstructed
request URLs and reported client addresses.

### Preserve duplicate cookies

Cookie helpers retain same-name cookies in order. `get(name)` returns the
first, `getAll(name)` returns all, and `append(name, value)` adds another.
Do not collapse multipart part headers into a native `Headers`; they are
lower-case-keyed plain objects.

### Choose origin protection and CSRF independently

Use `cop(options)` for browser-origin request protection. Use
`csrf(options)` plus `getCsrfToken(context)` for session-backed CSRF. They can
run independently or be layered with session middleware.

## Verification checklist

- Confirm imports use current domain subpaths.
- Confirm every middleware branch returns a response or `next()`.
- Confirm mounted features do not assume they own dispatch.
- Confirm route methods, generated URLs, and action maps agree.
- Confirm asynchronous UI updates check their abort signal.
- Confirm server boundaries catch `router.fetch()` errors and recognize
  request cancellation as `AbortError`.
- Confirm proxy trust matches the actual deployment boundary.
- Confirm cookie and multipart code preserves the new header semantics.
- Confirm data-table terminal operations are executed or bound as intended.
- Confirm tests and hooks consume `{ timeout, signal }` when they need
  cancellation behavior.

