# Route matching, parameters, and route errors

## Deterministic segment matching

Route matching traverses a segment trie rather than sorting and scanning a flat
route list. Ambiguous branches are explored by segment priority:

- fully static branches can win immediately;
- dynamic and optional branches are considered in deterministic priority order;
- wildcard branches are considered last.

Do not rely on file discovery order, route declaration order, or browser sort
behavior to decide among ambiguous candidates.

## Parameter priority

Set `params.priority` on routes that remain competing candidates after normal
segment matching. It is a tie-breaker, not a substitute for an unambiguous route
shape.

## Rejecting a candidate during parameter parsing

As an experimental behavior, `params.parse` may return `false` to reject an
incoming route candidate and let matching continue. A thrown parse error has a
different meaning: if that route is selected, the error surfaces on the match.

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

This candidate-rejection path applies to incoming URL matching. Outgoing typed
links made from a route template still perform exact route lookup and then call
`params.stringify`; they do not search for a different candidate based on
`params.parse`.

## Component-thrown not-found errors

A component may throw `notFound()` without an explicit `routeId`. The route's
`notFoundComponent` can handle it, and framework error boundaries preserve the
not-found error rather than converting it to a generic error.

```tsx
function Invoice() {
  const invoice = useInvoice()
  if (!invoice) throw notFound()
  return <InvoiceView invoice={invoice} />
}
```

## Primitive values thrown from `beforeLoad`

Router error handling preserves primitive values thrown from `beforeLoad`.
Consumers that inspect route errors must therefore not assume every caught value
is an `Error` instance.
