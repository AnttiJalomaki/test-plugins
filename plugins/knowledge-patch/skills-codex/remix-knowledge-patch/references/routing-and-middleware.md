# Routing and middleware

## Separate route definitions from handlers

The published Fetch router demonstrated in `remix-jam-2025` is built on the
Web Fetch API. A named route tree remains separate from its handlers, while
parameters and named splats are inferred. Middleware composes with
`router.use()`, and typed branches attach with `router.map()`:

```ts
import {
  createRouter,
  route,
  type RouteHandlers,
} from "@remix-run/fetch-router";

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

The beta uses domain-oriented `remix/fetch-router`, `remix/router`, and
`remix/routes` subpaths. Use the old package name only when maintaining code
from the earlier demonstration.

## Bind methods and generate URLs

A string leaf passed to `route()` creates a `Route<"ANY", Pattern>`. An object
leaf `{ method, pattern }` binds registration to a single method, and
`router.map()` honors it. `router.get()`, `router.post()`, and the other
method helpers constrain an `ANY` leaf when registering it.

Every leaf exposes `href()`:

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

Use generated URLs for links and form actions so their parameter types stay
coupled to the route definition.

## Generate conventional route maps

`form(path)` generates:

- `index`: `GET` at the supplied URL.
- `action`: `POST` at the same URL.

`resources(path)` generates `index`, `new`, `show`, `create`, `edit`, `update`,
and `destroy` using conventional `GET`, `POST`, `PUT`, and `DELETE` collection
and member paths. Singleton `resource(path)` omits `index`.

Both resource helpers accept `only`, `exclude`, custom `names`, and `param`:

```ts
resources("users", {
  only: ["index", "show"],
  param: "userId",
});
```

## Mount reusable installers

`router.mount()` accepts an installer under a parent pattern. This lets a
feature define local routes and be mounted at `/admin`, `/orgs/:orgId`, or
another prefix.

- Mount parameters reach child handlers.
- When parameter names repeat, the right-most value wins.
- The installer receives `RouteBuilder`, not the full `Router`.
- The parent retains request dispatch, matching, and the default handler.
- Unmatched paths below the prefix fall through to the parent unless the
  installer registers a catch-all.

The public composition types are `Router`, `RouteBuilder`, `RouteInstaller`,
`Action`, and `Controller`. `MapTarget` and `MapHandler` are no longer
exported.

## Always continue middleware explicitly

Middleware used by either router must return a `Response` or call and return
`next()`. An `undefined` fallthrough throws:

```ts
function loadUser(): Middleware {
  return (context, next) => {
    context.set(CurrentUser, user);
    return next();
  };
}
```

Middleware supplied to `createRouter({ middleware })` runs before matching on
every request. A `{ middleware, handler }` object on `router.map()` or a method
helper applies the chain only to that action.

Stored handlers retain route and middleware types through
`createAction(route, options)` and `createController(routeMap, options)`.

## Define the context contract

Built-in middleware installs direct properties such as:

- `context.auth`
- `context.formData`
- `context.logger`
- `context.render`
- `context.session`

Keyed `context.get(...)` access is still supported. Configure
`RouterTypes.context` once, then derive transformed contexts with
`RouterContext`, `MiddlewareContext`, or `ContextWithEntry`. Older auth,
renderer, async-context, and low-level transform helper types were removed.

Inline middleware arrays infer the stored action and controller context. Use
`createMiddleware()` only when inference crosses an exported chain, factory
return, or standalone `MiddlewareContext` derivation:

```ts
const protectedMiddleware = [requireAuth<AuthIdentity>()] as const;
type AppAuthContext =
  MiddlewareContext<typeof protectedMiddleware, AppContext>;
```

## Expose a keyed entry as a property

A provider can expose both a `createContextKey<T>()` entry and a direct
property. Declare the transform and repeat `{ property }` in `context.set()`:

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

Duplicate direct-property names throw. A key absent from the declared context
type and lacking a default may return `undefined`.

## Split route-pattern operations by purpose

The `remix/route-pattern` entrypoint focuses on parsing and serialization.
Use subpaths for:

- `/href`: URL generation.
- `/match`: matching.
- `/join`: joining.
- `/specificity`: ranking.

`match` and `matchAll` rank most-specific-first and no longer accept
`compareFn`. Sort returned matches explicitly for another order:

```ts
import * as Specificity from "remix/route-pattern/specificity";

matcher.matchAll(url).sort(Specificity.ascending);
```

## Map application actions

Generated apps, `remix doctor`, and `remix routes` expect controller-shaped
action files under `app/actions`, rather than `app/controllers`.

- Put root actions in `app/actions/controller.tsx`.
- Add explicit `router.map(...)` calls for nested route maps in
  `app/router.ts`.
- Controller middleware applies only to that controller's direct actions.

Do not assume controller middleware automatically wraps nested controllers or
mounted features.

## Catch dispatch errors at the server boundary

Errors from `router.fetch()` escape to its caller instead of becoming an
implicit error response. Catch and translate them at the server boundary.
An aborted request arrives as an `AbortError`; preserve that distinction when
logging or mapping errors.
