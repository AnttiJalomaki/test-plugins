# Apollo Client migration and runtime APIs

## Package, runtime, and constructor migration

### Public package boundaries (`client-v4-migration`)

Apollo Client 4 adds `rxjs` as a peer dependency. React APIs move to
`@apollo/client/react`, and `MockedProvider` moves to
`@apollo/client/testing/react`. Import only public entry points; direct `.js` and
`.cjs` paths are unsupported.

```sh
npm install @apollo/client@latest graphql rxjs
npx @apollo/client-codemod-migrate-3-to-4 src
```

The codemod handles many imports, links, removals, and client setup changes, but
review its output for application-specific behavior.

### Explicit transport and renamed options

The `uri`, `headers`, and `credentials` `ApolloClient` constructor shortcuts are
removed. Create an `HttpLink`. Move `name` and `version` under `clientAwareness`,
replace `connectToDevTools` with `devtools.enabled`, and rename
`disableNetworkFetches` to `prioritizeCacheValues`.

```ts
new ApolloClient({
  cache,
  link: new HttpLink({ uri, headers, credentials: "include" }),
  clientAwareness: { name: "web", version: "1.0.0" },
  devtools: { enabled: true },
  prioritizeCacheValues: true,
});
```

The package targets environments available since 2023 and Node.js 20+, supplies no
polyfills, and primarily selects development behavior with package export
conditions. Older targets must transpile or polyfill it themselves.

### React compatibility and removed APIs (`4.0.0`)

React 16 and `graphql` 15 are unsupported. The `Query`, `Mutation`, and
`Subscription` render-prop components, React HOCs, and `ApolloConsumer` are
removed. Use hooks and `useApolloClient`.

## Errors and policies

### Unified error model

Results expose `error`, not `errors`, and `ApolloError` is removed. Use guards such
as `CombinedGraphQLErrors.is(error)` and `CombinedProtocolErrors.is(error)`.
Network errors pass through unchanged; unusual thrown values become
`UnconventionalError`. `ServerError` and `ServerParseError` are classes with `.is`
guards, and `ServerError.bodyText` replaces its parsed `result`.

```ts
if (CombinedGraphQLErrors.is(error)) {
  for (const graphQLError of error.errors) handle(graphQLError);
}
```

Network errors now obey `errorPolicy`: `none` rejects, `all` resolves with
`result.error`, and `ignore` resolves without the error. `ObservableQuery` normally
delivers failures in `next` instead of terminating through the observer's `error`
callback. Subscription GraphQL errors behave likewise, although a nonrecoverable
network failure can terminate a subscription.

### Link error callbacks (`4.0.0`)

`ErrorLink` receives one `error` property rather than separate GraphQL, network,
and protocol error values. Classify it with the combined-error guards.
`LinkError.is(error)` identifies failures originating in the link chain.

```ts
const errorLink = new ErrorLink(({ error }) => {
  if (CombinedGraphQLErrors.is(error)) handleGraphQLErrors(error.errors);
});
```

### Mutation callback and result behavior (`3.13.0`)

If a `useMutation` `onCompleted` callback throws, the mutation promise rejects.
That callback failure is not passed to `onError`; handle it through the returned
promise.

`useMutation.ignoreResults` is deprecated and can cause extra rerenders after
removal. For a mutation that should not synchronize result state into a component, call
`client.mutate`:

```ts
const client = useApolloClient();
await client.mutate({ mutation, variables });
```

### Query lifecycle callbacks (`3.13.0`)

`onCompleted` and `onError` on `useQuery` and `useLazyQuery` are deprecated. New
code should derive behavior from result state or explicit promises instead of
depending on these lifecycle callbacks.

## Watched and lazy queries

### Watched-query lifecycle

In Client 4, `notifyOnNetworkStatusChange` defaults to `true`, and an uncached
`ObservableQuery` immediately emits a loading result. A query is tracked only while
it has a subscriber:

- `"active"` and `"all"` refetches exclude unobserved queries.
- Named standby queries are now refetched.
- From 4.0.11, a query skipped with `skipToken` is excluded until its first
  execution supplies variables.

`ObservableQuery.setOptions()` is removed; use public `reobserve()`.
`ObservableQuery.result()` is removed; convert it through RxJS.
Call `firstValueFrom()` on that conversion.

```ts
import { firstValueFrom, from } from "rxjs";

await observable.reobserve(newOptions);
const result = await firstValueFrom(from(observable));
```

### Safe `updateQuery` previous data (`3.13.0`)

The first previous-data argument to `ObservableQuery.updateQuery` is deprecated
because it can be partial while typed as complete. Use `previousData` and
`complete` from the second argument. Returning `undefined` is explicitly supported
to skip an update.

```ts
observableQuery.updateQuery(
  (_unsafe, { previousData, complete }) =>
    complete ? previousData : undefined
);
```

`subscribeToMore` now types the callback's second-argument `variables` as the
parent query variables, not the subscription variables.

### Execute-driven `useLazyQuery`

Changing hook options no longer starts a request. Put `variables` and `context` on
the execute call and other options on the hook. Do not execute during render or
SSR. Unmounting or a newer execution aborts the in-flight call with `AbortError`;
call `.retain()` only when the request must finish.

```ts
const [execute] = useLazyQuery(QUERY, { fetchPolicy: "no-cache" });
const result = await execute({ variables, context }).retain();
```

### `fetchMore` option independence (`4.0.0`)

`fetchMore` defaults its own `errorPolicy` to `none`; set another policy when
partial pagination data is acceptable. Without a replacement query, variables
are shallow-merged. With a replacement query, supplied variables are used as-is.
Calling `fetchMore` for a `cache-only` query throws.

```ts
await observable.fetchMore({
  variables: { offset: nextOffset },
  errorPolicy: "all",
});
```

## RxJS and subscriptions

Client 4 replaces `zen-observable` with RxJS. `ObservableQuery` implements only
`Subscribable`; wrap it with `from(observableQuery)` where an RxJS `Observable` is
required, and replace old instance operators with `pipe` operators.

Subscriptions are deduplicated by default. A subscriber joining an existing
connection does not receive that connection's initial server value. Disable
deduplication per subscription with
`context: { queryDeduplication: false }` when that replay difference matters.

`client.subscribe()` is lazy: no connection opens until subscription. Its returned
observable has `restart()`, which tears down and recreates its link request.

```ts
const observable = client.subscribe({ query: SUBSCRIPTION });
const subscription = observable.subscribe({ next: handleValue });
observable.restart();
```

## Links and GraphQL-over-HTTP

### Class-based links

Prefer `HttpLink`, `SetContextLink`, `ErrorLink`, `PersistedQueryLink`, and
`RemoveTypenameFromVariablesLink` over creator functions. `SetContextLink` reverses
the legacy callback parameters to `(previousContext, operation)`.

Use `ApolloLink` static composition methods. Static `concat` is itself deprecated
in favor of `from`; `from`, `concat`, and `split` require `ApolloLink` instances,
so wrap bare handlers.

### Custom operation context

`operation.getContext()` is frozen; update context with `setContext()`. Use
`operation.client.cache` instead of removed context members `cache` and
`getCacheKey`. Inspect `operation.operationType` instead of parsing the document.
When executing a link directly, pass the client:

```ts
execute(link, request, { client });
```

### Strict response media handling

HTTP links advertise
`application/graphql-response+json,application/json;q=0.9` and interpret status
codes according to response media type. A non-200 `application/json` response is a
`ServerError`; mocks should use the production `Content-Type`.

### Enhanced client awareness (`4.0.0`)

`HttpLink` and `BatchHttpLink` default `includeExtensions` to `true` and send
`extensions.clientLibrary`. Disable transport metadata when it should not leave
the client:

```ts
new HttpLink({
  uri: "/graphql",
  enhancedClientAwareness: { transport: false },
});
```

## Type migration

Types are namespaced with their owning APIs, such as `ApolloLink.Result`,
`ApolloClient.Options`, `ObservableQuery.Result`, and `useQuery.Options`. The
`TContext` generic is removed; augment `DefaultContext`. The `ApolloClient`
cache-shape and `ApolloCache` serialization generics are also removed.

```ts
import "@apollo/client";

declare module "@apollo/client" {
  interface DefaultContext {
    requestId?: string;
  }
}
```

From Client 4.2, when `defaultOptions` specifies `errorPolicy` or
`returnPartialData`, declare the defaults under
`ApolloClient.DeclareDefaultOptions` so result types reflect them. A required
declaration chooses document-inferred signatures that do not take manually
supplied result generics.

```ts
declare module "@apollo/client" {
  namespace ApolloClient.DeclareDefaultOptions {
    interface WatchQuery {
      errorPolicy: "all";
    }
  }
}
```

## SSR and tests

### Static rendering (`4.0.0`)

`prerenderStatic` replaces `getDataFromTree`, `getMarkupFromTree`, and
`renderToStringWithData`; it supports Suspense hooks with React 19 static
rendering.

```ts
import { prerenderStatic } from "@apollo/client/react/ssr";

const html = await prerenderStatic(<App />, { client });
```

### Mock matching and timing

`MockLink` removes `variableMatcher`; use a predicate in `request.variables`
(`4.0.0`):

```ts
new MockLink([{
  request: {
    query: QUERY,
    variables: variables => variables.id !== undefined,
  },
  result: { data: mockData },
}]);
```

Mocks without an explicit delay have a random 20–50 ms delay so tests exercise
loading states. Set `MockLink.defaultOptions = { delay: 0 }` only when globally
immediate results are intentional.

## Preparing on Client 3 (`3.14.0`)

Client 3.14 emits migration warnings and deprecation annotations for Client 4
changes. Remove `addTypename` configuration from `InMemoryCache` and
`MockedProvider`, remove `canonizeResults`, and do not use `standby` with
`client.query`.

Replace `queryRef.toPromise()` with the function owned by the preloader:

```ts
const result = await preloadQuery.toPromise(queryRef);
```
