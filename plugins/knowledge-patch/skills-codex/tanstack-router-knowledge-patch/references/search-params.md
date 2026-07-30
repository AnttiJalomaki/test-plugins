# Search parameters

## Validation failure flow

`validateSearch` receives the JSON-parsed but still unvalidated search object.
If validation throws, the route's `onError` hook runs with
`error.routerCode === 'VALIDATE_SEARCH'`, then the route renders its
`errorComponent`.

Use tolerant defaults when an externally edited or malformed URL should remain
navigable:

```tsx
const searchSchema = z.object({
  page: z.number().catch(1),
  sort: z.enum(['newest', 'oldest']).catch('newest'),
})

export const Route = createFileRoute('/products')({
  validateSearch: searchSchema,
})
```

Reserve thrown validation errors for URLs that should interrupt navigation and
enter the route error flow.

## Validator input and output types

Search validators have two important types:

- navigation is checked against the validator's input type;
- `Route.useSearch()` and other reads expose its output type.

Defaults make search fields optional at navigation sites only if the integration
preserves both types. For Zod v3, use `@tanstack/zod-adapter`; use its `fallback`
helper instead of a Zod v3 `.catch()` that erases the needed input/output
distinction.

```tsx
import { fallback, zodValidator } from '@tanstack/zod-adapter'

const searchSchema = z.object({
  page: fallback(z.number(), 1).default(1),
})

export const Route = createFileRoute('/products')({
  validateSearch: zodValidator(searchSchema),
})

// The default makes search optional for this navigation.
const link = <Link to="/products" />
```

Zod v4 schemas can be supplied directly. Standard Schema implementations can
also be supplied directly, including Valibot 1, ArkType 2, and Effect's
`standardSchemaV1`.

## Search middlewares

Configure route-level search transforms under `search.middlewares`. A route's
middlewares apply while constructing links to that route or its descendants.
They run again during navigation, after validation.

Middlewares compose through `next`. The built-in helpers cover common URL
policy:

- `retainSearchParams` carries selected values from the current search into the
  next location;
- `stripSearchParams` removes values equal to supplied defaults, keeping the
  visible URL compact.

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

Because these middlewares affect descendant links as well as the route itself,
place them at the narrowest route whose descendants share the same search
policy.
