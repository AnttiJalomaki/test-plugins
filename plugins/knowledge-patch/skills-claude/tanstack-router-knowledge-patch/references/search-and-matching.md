# Search, Matching, Parameters, and Route Errors

## Search validation failures

`validateSearch` receives a JSON-parsed but otherwise unvalidated search
object. If validation throws, the route's `onError` runs with
`error.routerCode === 'VALIDATE_SEARCH'`, then the route renders its
`errorComponent`. Use tolerant schema fallbacks when malformed shared URLs
should recover rather than interrupt navigation.

```tsx
const searchSchema = z.object({
  page: z.number().catch(1),
  sort: z.enum(['newest', 'oldest']).catch('newest'),
})

export const Route = createFileRoute('/products')({
  validateSearch: searchSchema,
})
```

## Validator input and output types

Navigation is typed from a validator's input, whereas reading route search uses
its output. Defaults make a search field optional for `<Link>` and `navigate`
only if the validator integration preserves both types.

Zod v3 requires `@tanstack/zod-adapter`. Use its `zodValidator` and `fallback`
helper rather than a Zod v3 `.catch()` that erases the distinct input type:

```tsx
import { fallback, zodValidator } from '@tanstack/zod-adapter'

const searchSchema = z.object({
  page: fallback(z.number(), 1).default(1),
})

export const Route = createFileRoute('/products')({
  validateSearch: zodValidator(searchSchema),
})

// The default makes search optional.
const link = <Link to="/products" />
```

Zod v4 schemas can be passed directly. Standard Schema implementations,
including Valibot 1, ArkType 2, and Effect's `standardSchemaV1`, can also be
used directly.

## Search middleware

A route's `search.middlewares` transform search when constructing links to that
route or any descendant. They run again on navigation after validation.
Middleware functions compose through `next`.

Use `retainSearchParams` to carry selected values from the current search and
`stripSearchParams` to omit values that equal supplied defaults:

```tsx
import {
  createFileRoute,
  retainSearchParams,
  stripSearchParams,
} from '@tanstack/react-router'

export const Route = createFileRoute('/search')({
  validateSearch: searchSchema,
  search: {
    middlewares: [
      retainSearchParams(['campaign']),
      stripSearchParams({ page: 1, tags: [] }),
    ],
  },
})
```

Order middlewares according to the transformation desired both when links are
created and after validated navigation.

## Deterministic segment-priority matching

Route matching traverses a segment trie rather than sorting and scanning a flat
route list. When branches are ambiguous, it explores static, dynamic, optional,
and wildcard segments by priority. A fully static branch can win immediately;
wildcards are considered last. Matching therefore does not depend on browser
sorting behavior.

Routes may set `params.priority` as a tie-breaker when otherwise competing
route candidates remain.

## Rejecting candidates from parameter parsing

`params.parse` can experimentally return `false` to reject an incoming route
candidate and allow matching to continue. Throwing from `parse` still exposes
the parse error on the selected match.

Outgoing typed links to a route template are different: Router performs exact
route lookup and then calls `params.stringify`.

```tsx
const reportRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/reports/$id',
  params: {
    parse: (raw) =>
      /^\d+$/.test(raw.id) ? { id: Number(raw.id) } : false,
    stringify: ({ id }) => ({ id: String(id) }),
  },
})
```

Because returning `false` is experimental, keep tests for competing route
candidates and invalid parameter shapes.

## Route-level error preservation

A component can throw `notFound()` without an explicit `routeId`. Its route's
`notFoundComponent` handles the error, and framework error boundaries preserve
it.

Primitive values thrown from `beforeLoad` are likewise preserved through
Router's error handling. Do not assume route errors are always `Error`
instances when inspecting or rendering them.
