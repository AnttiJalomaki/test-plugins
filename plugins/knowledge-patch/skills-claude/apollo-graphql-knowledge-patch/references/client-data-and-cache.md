# Apollo Client data, cache, and refetch behavior

## Result completeness and Suspense

### `dataState` (`4.0.0`)

`ObservableQuery` and data-returning React hooks expose:

- `empty`: `data` is `undefined`.
- `partial`: partial cache data is present with `returnPartialData`.
- `streaming`: an incremental result is unfinished.
- `complete`: data is fully satisfied.

During `@defer` streaming, `loading` remains `true` and `networkStatus` is
`NetworkStatus.streaming`.

```ts
const { data, dataState } = useQuery(QUERY, { returnPartialData: true });
if (dataState === "complete") renderCompleteResult(data);
```

### Suspense fragments (`3.13.0`)

`useSuspenseFragment` is a drop-in `useFragment` replacement that suspends until
the requested fragment is complete, leaving its Suspense boundary responsible for
loading UI.

For multipart queries such as `@defer`, query deduplication remains active until
the final response chunk, not merely the first result.

## Incremental delivery

### Opt-in handlers (`client-v4-migration`)

Client 4 requires `incrementalHandler`; it initially supports the 2022-08-24
protocol through `Defer20220824Handler`. Enable custom-link incremental result
types and GraphQL Code Generator masking types with `TypeOverrides`.

```ts
import { Defer20220824Handler } from "@apollo/client/incremental";
import { GraphQLCodegenDataMasking } from "@apollo/client/masking";

const client = new ApolloClient({
  cache,
  link,
  incrementalHandler: new Defer20220824Handler(),
});

declare module "@apollo/client" {
  interface TypeOverrides
    extends Defer20220824Handler.TypeOverrides,
      GraphQLCodegenDataMasking.TypeOverrides {}
}
```

### GraphQL.js alpha 9 (`4.1.0`)

Use `GraphQL17Alpha9Handler` only for servers that implement the GraphQL.js 17
alpha 9 incremental format. Older formats still require their matching handler.

```ts
const client = new ApolloClient({
  cache,
  link,
  incrementalHandler: new GraphQL17Alpha9Handler(),
});
```

`Defer20220824Handler` and `GraphQL17Alpha2Handler` support `@stream`. Encountering
`@stream` without an incremental handler throws. The older defer protocol lacks
stream metadata and truncates the existing array on the first chunk; the
stream-aware default merge truncates on the final chunk.

## Fragment sources

`useFragment`, `useSuspenseFragment`, and `client.watchFragment` accept an array in
`from` and return index-aligned data (`4.1.0`). Fragment watches accept
`from: null`, producing `{ data: null, dataState: "complete", complete: true }`.
`readFragment`, `watchFragment`, and `updateFragment` expose `from`.

```ts
const { data } = useFragment({
  fragment: ItemFragment,
  from: [firstItem, secondItem],
});
```

`preloadQuery.toPromise(queryRef)` is the supported promise conversion; the
`queryRef.toPromise()` method was deprecated in `3.14.0` and removed in Client 4.

From `4.2.0`, preloaded query references incorporate declared watch-query defaults.
For example, a declared `errorPolicy: "all"` yields
`PreloadedQueryRef<TData, TVariables, "complete" | "streaming" | "empty">`.

## Cache reads, writes, and policies

### Extensions in merge functions (`4.1.0`)

`cache.write`, `cache.writeQuery`, and `client.writeQuery` accept `extensions`.
These values and extensions received from GraphQL operations are forwarded to
field-policy `merge` options.

```ts
client.writeQuery({
  query: ITEMS_QUERY,
  data: { items },
  extensions: { source: "prefetch" },
});
```

### Partial arrays

`InMemoryCache` preserves `undefined` items returned by an array field's `read`
function. The array can therefore represent partial data and trigger a network
fetch for the complete list.

### Supertype policy isolation (`3.14.0`)

Subtype field policies no longer overwrite or merge into policies declared on a
supertype. Configuring a subtype cannot mutate the inherited policy definition.

### Local fields and custom caches

Using `@client` in Client 4 requires `new LocalState({ resolvers })` in the
client's `localState` option. Resolver context is
`{ requestContext, client, phase }`. Normal resolution converts `undefined` to
`null` with a warning; non-scalar objects need `__typename`; thrown resolver
failures become GraphQL errors; and `@export` needs a matching variable definition
with values for required variables.

A custom cache must implement `fragmentMatches`; `InMemoryCache` already does.
Without it, `LocalState` throws.

When a cache read handles an `@client` field, `existing` is `undefined` rather than
forced `null`, so default parameters work (`4.1.0`). A custom `ApolloCache` can
return `true` from `resolvesClientField` to tell `LocalState` it supplies a field.
Returning `false` or omitting the method produces a warning and `null`.

### Fully skipped operations (`4.1.0`)

When every field is skipped, query data is `{}` rather than `null`. This also
prevents `useSuspenseQuery` from suspending indefinitely.

## Mutations and typed defaults

### Context callback (`4.1.0`)

The mutate function returned by `useMutation` accepts `context` as a callback. It
receives hook-level context so call-specific values can extend defaults.

```ts
const [mutate] = useMutation(SAVE_ITEM, {
  context: { trace: true },
});

await mutate({
  context: hookContext => ({ ...hookContext, urgent: true }),
});
```

### Signature style (`4.2.0`)

Set `TypeOverrides.signatureStyle` to `"modern"` for document-inferred,
default-aware signatures without declaring a required default option. `"classic"`
temporarily keeps generic-taking signatures, but declared defaults then stop
affecting result types.

```ts
declare module "@apollo/client" {
  interface TypeOverrides {
    signatureStyle: "modern";
  }
}
```

For multiple clients with conflicting defaults, declare a narrow union under
`DeclareDefaultOptions.WatchQuery`. Optional `errorPolicy` also includes runtime
default `"none"` in inferred results.

```ts
declare module "@apollo/client" {
  namespace ApolloClient.DeclareDefaultOptions {
    interface WatchQuery {
      errorPolicy?: "all" | "ignore";
    }
  }
}
```

`client.query` data is non-nullable when effective `errorPolicy` is `"none"`.
`ApolloClient.MutateResult<TData, TErrorPolicy>` maps:

- `"none"` to `data: TData` and no error.
- `"all"` to possibly undefined data and optional error.
- `"ignore"` to possibly undefined data and no error.

`client.mutate` and `useMutation` use the declared mutation default unless the
call overrides `errorPolicy`; `useMutation.Result.error` is `undefined` for
`"ignore"`.

## Event-driven refetching (`4.2.0`)

Automatic focus and connectivity refetching is opt-in through
`RefetchEventManager`. Built-in `windowFocusSource` and `onlineSource` events
refetch active queries by default.

```ts
const manager = new RefetchEventManager({
  sources: {
    windowFocus: windowFocusSource,
    online: onlineSource,
  },
});

const client = new ApolloClient({
  cache,
  link,
  refetchEventManager: manager,
  defaultOptions: { watchQuery: { refetchOn: false } },
});

useQuery(DASHBOARD_QUERY, {
  refetchOn: { windowFocus: true },
});
```

`refetchOn` accepts a boolean, a per-event map, or a predicate receiving source and
payload. Per-query maps merge with `defaultOptions.watchQuery.refetchOn`; omitted
events continue to use the default boolean or predicate.

Augment `RefetchEvents` to define custom payloads, supply an Observable source, or
register `true` for an imperative-only event and call `emit`.

```ts
declare module "@apollo/client" {
  interface RefetchEvents {
    userTriggered: void;
  }
}

const events = new RefetchEventManager({
  sources: { userTriggered: true },
});
events.emit("userTriggered");
```

Per-event `handlers` can replace normal active-query refetching and receive
`matchesRefetchOn` for applying query settings. `defaultHandler` or
`setDefaultEventHandler` changes the fallback. A handler returns
`RefetchQueriesResult`, or `void` to skip.

## Awareness and static rendering (`4.1.0`)

Enhanced client awareness can travel through headers as well as request extensions.

`prerenderStatic` returns the value from its `renderFunction` and reports `aborted`
correctly, supporting React 19.2 `resumeAndPrerender` flows.
