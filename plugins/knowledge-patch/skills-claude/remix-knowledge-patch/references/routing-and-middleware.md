# Routing and middleware

Use this reference for named route trees, route-map shorthands, mounted feature
installers, typed middleware context, route-pattern helpers, and server-boundary
error handling.

## Fetch router model

`@remix-run/fetch-router` was already published when the Remix Jam component
model was demonstrated. It is a type-safe router built on the Web Fetch API.
The named route tree is separate from handlers, parameters and named splats are
typed, `router.use()` composes middleware, and `router.map()` maps a typed branch.
Source batch: `remix-jam-2025`.

```ts
import {
  createRouter,
  route,
  type RouteHandlers,
} from "@remix-run/fetch-router";
import { uploadHandler } from "./uploads";

const routes = route({
  uploads: "/uploads/*key",
  books: { show: "/books/:slug" },
});

const books = {
  handlers: {
    show({ params }) {
      return new Response(params.slug);
    },
  },
} satisfies RouteHandlers<typeof routes.books>;

const router = createRouter({ uploadHandler });
router.map(routes.books, books);
```

For current beta code, prefer the domain subpaths such as `remix/router` and
`remix/routes`; the older package form above documents the published preview
at the time of that demonstration.

## Method-bound leaves and URL generation

A string leaf passed to `route()` creates `Route<"ANY", Pattern>`. A leaf of
the form `{ method, pattern }` binds mapping to a single HTTP method.

`router.map()` honors a bound method. When a leaf is `ANY`, method helpers such
as `router.get()` and `router.post()` constrain it during registration. Every
route leaf provides `href()` for typed links and form actions.

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

## Form and REST-style shorthands

`form(path)` creates two actions at the same URL:

- `index` with `GET`; and
- `action` with `POST`.

`resources(path)` creates conventional collection and member actions:

- `index` and `new` with `GET` collection paths;
- `show` and `edit` with `GET` member paths;
- `create` with `POST`;
- `update` with `PUT`; and
- `destroy` with `DELETE`.

`resource(path)` creates a singleton and omits `index`. Both resource helpers
accept `only`, `exclude`, custom `names`, and `param`:

```ts
const userRoutes = resources("users", {
  only: ["index", "show"],
  param: "userId",
});
```

## Mount reusable route installers

`router.mount(pattern, installer)` installs a reusable route group below a
parent pattern such as `/admin` or `/orgs/:orgId`. Parameters from the mount
reach child handlers. If parent and child use the same parameter name, the
right-most value wins.

An installer receives a `RouteBuilder`, not a full `Router`. It can register
routes, but the parent retains request dispatch, matching, and the default
handler. Consequently, an unmatched path below the prefix falls through to the
parent unless the installer registers its own catch-all.

The public composition types are `Router`, `RouteBuilder`, `RouteInstaller`,
`Action`, and `Controller`. `MapTarget` and `MapHandler` are no longer exported.

## Middleware continuation is explicit

Middleware used by `remix/router` or `remix/fetch-router` must do one of two
things on every path:

- return a `Response`; or
- call and return `next()`.

Falling through with `undefined` throws instead of implicitly continuing.

```ts
function loadUser(): Middleware {
  return (context, next) => {
    context.set(CurrentUser, user);
    return next();
  };
}
```

## Global and per-action middleware

Middleware supplied to `createRouter({ middleware })` runs before matching on
every request. A mapping or method helper accepts `{ middleware, handler }` to
attach a chain to one action.

Stored handlers preserve route and middleware types when built through
`createAction(route, options)` and `createController(routeMap, options)`.
Although generated application files moved from controllers to actions, the
typed `Controller` composition concept and `createController` API remain part
of router composition.

## Built-in context typing

Built-in middleware installs direct properties including:

- `context.auth`;
- `context.formData`;
- `context.logger`;
- `context.render`; and
- `context.session`.

Keyed `context.get(...)` access is still supported. Configure
`RouterTypes.context` once for the application's base context. Derive contexts
after transforms with `RouterContext`, `MiddlewareContext`, or
`ContextWithEntry`.

The older authentication, renderer, async-context, and low-level context
transform helper types were removed.

Inline middleware arrays infer the stored action or controller context. Use
`createMiddleware()` when inference must cross a boundary, such as:

- an exported middleware array;
- a chain returned from a factory; or
- a standalone `MiddlewareContext` derivation.

```ts
const protectedMiddleware = [requireAuth<AuthIdentity>()] as const;
type AppAuthContext = MiddlewareContext<
  typeof protectedMiddleware,
  AppContext
>;
```

## Expose custom context properties

A provider can expose both a `createContextKey<T>()` entry and a direct
property. Declare the key, value, and property in the middleware transform and
repeat `{ property }` when setting it:

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

router.get("/books", {
  middleware: [loadDatabase()],
  handler: async ({ db }) => Response.json(await db.findMany()),
});
```

Duplicate direct properties throw. A key can produce `undefined` if it is not
present in the declared context type and has no default.

## Route-pattern subpaths and ranking

The base `remix/route-pattern` entrypoint focuses on parsing and serialization.
Use purpose-specific subpaths for operations:

| Operation | Subpath |
| --- | --- |
| URL generation | `remix/route-pattern/href` |
| Matching | `remix/route-pattern/match` |
| Joining | `remix/route-pattern/join` |
| Ranking | `remix/route-pattern/specificity` |

`match()` and `matchAll()` always rank most-specific-first and no longer accept
`compareFn`. Sort `matchAll()` results after matching to apply another order:

```ts
import * as Specificity from "remix/route-pattern/specificity";

matcher.matchAll(url).sort(Specificity.ascending);
```

## Actions and route maps

Generated application code uses `app/actions` instead of `app/controllers`.
Root actions live in `app/actions/controller.tsx`. Map nested route maps
explicitly with `router.map(...)` in `app/router.ts`; controller middleware
applies only to the controller's direct actions.

## Dispatch errors

`router.fetch()` lets thrown errors escape to its caller rather than converting
them into an implicit error response. Catch them at the server boundary and
translate them there if the deployment requires an HTTP response. An aborted
request arrives as `AbortError`, so preserve abort semantics instead of treating
it as an ordinary application failure.
