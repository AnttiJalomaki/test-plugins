---
name: remix-knowledge-patch
description: Remix
version: 3.0.0-beta.5
license: MIT
metadata:
  author: Nevaberry
---


# Remix Knowledge Patch

Use this skill when designing, migrating, reviewing, or debugging Remix 3
applications and packages. Start by identifying which product line the project
actually uses: the incremental React path is React Router v7 framework mode,
whereas Remix 3 is a separate, non-React framework under active development.

## Reference index

| Reference | Topics |
| --- | --- |
| [architecture-and-migration.md](references/architecture-and-migration.md) | Product-line choice, framework goals, beta status, package imports, scaffolding, and repository previews |
| [routing-and-middleware.md](references/routing-and-middleware.md) | Typed route maps, mounting, resource routes, middleware context, route patterns, actions, and dispatch errors |
| [ui-and-frames.md](references/ui-and-frames.md) | Component handles, mixins, hydration, frames, popovers, DOM preservation, and generated CSS |
| [data-http-and-security.md](references/data-http-and-security.md) | Data tables, headers, multipart parsing, storage, authentication, origin protection, and trusted proxies |
| [runtime-cli-and-testing.md](references/runtime-cli-and-testing.md) | Portable runtime contracts, asset compilation, Node hosting, CLI requirements, loaders, and tests |

## Choose the product line before changing code

- Treat Remix v1/v2 continuation through React Router v7 framework mode as the
  stable React-based path.
- Treat Remix 3 as a new component model and package family, not as the renamed
  RSC-first successor once anticipated for React applications.
- For incremental React Server Components, evaluate React Router v7 support for
  server-component loader/action returns and server-only routes.
- Use Remix 3 betas for experiments and prototypes, not production deployment.
- Do not infer current Remix 3 APIs from an early demonstration. In particular,
  the public beta passes a `Handle` argument and composes behavior with `mix`.

See [architecture-and-migration.md](references/architecture-and-migration.md)
for the architectural split and migration-oriented package map.

## Breaking changes to address first

### Replace root and legacy package imports

There is no `remix` root export. Import from domain subpaths:

```ts
import { createRouter } from "remix/router";
import { requireAuth } from "remix/middleware/auth";
import { sql } from "remix/data-table";
import { on } from "remix/ui";
```

The temporary aliases corresponding to older `@remix-run/*` packages are
migration aids and are planned to disappear before stable. Prefer the domain
subpaths in new code.

### Return from every middleware path

Middleware must return a `Response` or return `next()`. Falling through with
`undefined` is an error:

```ts
function loadUser(): Middleware {
  return (context, next) => {
    context.set(CurrentUser, user);
    return next();
  };
}
```

### Move generated application controllers to actions

- Put generated app action files under `app/actions`, not `app/controllers`.
- Put root actions in `app/actions/controller.tsx`.
- Add explicit `router.map(...)` calls for nested route maps in `app/router.ts`.
- Remember that controller middleware wraps only that controller's direct
  actions.

### Update removed and relocated APIs

| Old assumption | Current action |
| --- | --- |
| `remix/component*` or `remix/components/*` | Use `remix/ui`, its JSX/server runtime subpaths, and `remix/ui/<component>` |
| `remix/data-table/sql` for `SqlStatement`, `sql`, `rawSql` | Import them from `remix/data-table` |
| `QueryBuilder` | Use `Query` and `query` |
| Route-pattern helpers from the base entrypoint | Use `/href`, `/match`, `/join`, or `/specificity` |
| `MapTarget` and `MapHandler` | Use the public router composition types |
| `app/controllers` | Use `app/actions` |
| `remix-test` executable | Run `remix test` |
| `remix/node-serve` | Choose another available server path until it returns |

The UI removals, data-query semantics, public router types, and loader behavior
are detailed in the topic references.

## Typed routing quick reference

Define a named route tree separately from its handlers. A string leaf accepts
any method until registration constrains it; `{ method, pattern }` binds the
method in the route definition. Every leaf has typed `href()` generation.

```ts
const routes = route({
  contact: {
    index: { method: "GET", pattern: "/contact" },
    action: { method: "POST", pattern: "/contact" },
  },
});

router.map(routes.contact, {
  actions: {
    index: () => new Response("Contact"),
    action: () => new Response("Sent"),
  },
});

const formAction = routes.contact.action.href();
```

Use `form(path)` for paired GET/POST form routes, `resources(path)` for a
conventional collection, and `resource(path)` for a singleton. Mount reusable
installers with `router.mount()`, but keep dispatch and the default handler on
the parent router.

See [routing-and-middleware.md](references/routing-and-middleware.md) before
customizing match order, exporting typed middleware chains, or relying on
mounted-prefix fallbacks.

## Middleware context quick reference

Global middleware supplied to `createRouter({ middleware })` runs before route
matching. Per-action middleware belongs beside `handler` in a mapping or method
registration. Built-in middleware adds direct context properties such as
`auth`, `formData`, `logger`, `render`, and `session`; keyed access remains
available.

For custom middleware, declare both the context-key transform and the direct
property, then pass the same property to `context.set()`:

```ts
const Database = createContextKey<Database>();

function loadDatabase(): Middleware<{
  key: typeof Database;
  value: Database;
  property: "db";
}> {
  return async (context, next) => {
    context.set(Database, await connectDatabase(), { property: "db" });
    return next();
  };
}
```

Inline middleware arrays usually infer context. Reach for `createMiddleware()`
when the chain crosses an inference boundary, such as an exported constant, a
factory result, or a standalone type derivation.

## UI quick reference

Components keep mutable local state and return a render callback. In the beta
API the handle is a normal argument, behavior is composed through `mix`, and an
async `on()` handler must honor its abort signal before changing state:

```tsx
import { type Handle, on } from "remix/ui";

function Copy(handle: Handle<{ url: string }>) {
  let copied = false;
  return () => (
    <button mix={[on("click", async (_, signal) => {
      await navigator.clipboard.writeText(handle.props.url);
      if (signal.aborted) return;
      copied = true;
      handle.update();
    })]}>
      {copied ? "Copied" : "Copy"}
    </button>
  );
}
```

Use frame APIs for asynchronous HTML-backed UI, `rmx-preserve-dom` for the
smallest live widget subtree that frame reconciliation must not replace, and
remember that `css(...)` output lives in the `rmx` cascade layer.

See [ui-and-frames.md](references/ui-and-frames.md) for navigation, SSR frame
sources, popover composition, right-click menus, and the removed UI exports.

## HTTP, data, and security quick reference

- Same-name cookies are ordered and preserved; use `getAll(name)` when every
  value matters.
- `MultipartPart.headers` is a lower-case-keyed plain object, not `Headers`.
- Data-table terminal calls can produce `Query` values; pass `Query` or raw SQL
  to `db.exec(...)`.
- Enable `trustProxy` only behind a trusted reverse proxy.
- Configure tokenless browser-origin protection with `cop(options)` and
  session-backed CSRF with `csrf(options)`; they are independent and composable.

See [data-http-and-security.md](references/data-http-and-security.md) for the
adapter inventory and precise header, query, and protection behavior.

## Runtime and tooling quick reference

- Prefer Fetch and browser-standard data types at package boundaries.
- Use `assets` for on-demand browser TypeScript, JavaScript, and CSS builds.
- Use `node-fetch-server` to host a Fetch server on Node.js.
- Use `node --import remix/node-tsx app.tsx` for Node TypeScript/JSX execution.
- The CLI metadata requires Node.js 24.3.0 or newer.
- Tests and hooks receive `{ timeout, signal }`; `t.signal` aborts on timeout.

See [runtime-cli-and-testing.md](references/runtime-cli-and-testing.md) for
repository preview installs, loader semantics, and server availability.

## Review checklist

1. Confirm whether the codebase is React Router framework mode or Remix 3.
2. Reject root imports and migrate temporary aliases toward domain subpaths.
3. Verify every middleware branch returns a response or `next()`.
4. Check mounted installers for an explicit catch-all when one is intended.
5. Catch `router.fetch()` failures at the server boundary, including aborts.
6. Validate async UI handlers against their abort signal before updating.
7. Treat multipart headers as plain objects and duplicate cookies as ordered.
8. Keep forwarded-header trust disabled unless the proxy boundary is trusted.
9. Check the Node version before diagnosing CLI or test-loader failures.
10. Keep beta prototypes away from production-critical deployment paths.
