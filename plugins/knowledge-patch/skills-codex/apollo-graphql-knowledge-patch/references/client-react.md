# Apollo Client and React

## Migration and package boundaries

### Prepare for Client 4

Apollo Client 3.14.0 annotates and warns about APIs that change or disappear in
Client 4. Deprecated configuration includes `addTypename` on `InMemoryCache`
and `MockedProvider`, `canonizeResults`, and use of `standby` with
`client.query`. Clear these warnings before upgrading.

For the client-v4-migration:

- Add the `rxjs` peer dependency.
- Import React APIs from `@apollo/client/react` and `MockedProvider` from
  `@apollo/client/testing/react`.
- Use public package entry points; remove direct `.js` and `.cjs` imports.
- Run `npx @apollo/client-codemod-migrate-3-to-4 src` for the mechanical import,
  link, removal, and setup changes.

The package targets environments available since 2023 and Node.js 20 or newer.
It ships no polyfills and selects most development behavior through package
export conditions. Transpile or polyfill for older targets.

### Client construction

The client-v4-migration removes the `uri`, `headers`, and `credentials`
constructor shortcuts. Build an `HttpLink` and pass it as `link`. Move `name`
and `version` to `clientAwareness`, rename `connectToDevTools` to
`devtools.enabled`, and rename `disableNetworkFetches` to
`prioritizeCacheValues`.

```ts
new ApolloClient({
  cache,
  link: new HttpLink({ uri, headers, credentials: "include" }),
  clientAwareness: { name: "web", version: "1.0.0" },
  devtools: { enabled: true },
  prioritizeCacheValues: true,
});
```

Client 4.0.0 no longer supports React 16 or `graphql` 15. It removes the
`Query`, `Mutation`, and `Subscription` render-prop components, React HOCs, and
`ApolloConsumer`; use hooks and `useApolloClient`.

## Query and fragment lifecycle

### Suspense and completeness

Since 3.13.0, `useSuspenseFragment` can replace `useFragment` when a Suspense
boundary should own loading; it suspends until the fragment is complete.

Client 4.0.0 adds `dataState` to `ObservableQuery` and data-returning hooks:

- `empty`: `data` is `undefined`.
- `partial`: partial cache data is returned.
- `streaming`: a deferred result is unfinished.
- `complete`: the result is fully satisfied.

While deferred data is streaming, `loading` remains `true` and
`networkStatus` is `NetworkStatus.streaming`.

### Query callbacks and previous data

The `onCompleted` and `onError` options on `useQuery` and `useLazyQuery` are
deprecated since 3.13.0. Do not base new query lifecycle logic on them.

Also since 3.13.0, the first previous-data parameter of
`ObservableQuery.updateQuery` is deprecated because it can be partial while
typed as complete. Read `previousData` and `complete` from the second argument.
Returning `undefined` explicitly skips the update.

```ts
observableQuery.updateQuery(
  (_unsafe, { previousData, complete }) =>
    complete ? previousData : undefined
);
```

The `variables` supplied to a `subscribeToMore` update callback are typed as
the parent query's variables, not the subscription variables (3.13.0).

### Watched queries and lazy execution

In the client-v4-migration, `notifyOnNetworkStatusChange` defaults to `true`,
and an uncached `ObservableQuery` immediately emits a loading result. A query is
tracked only while it has a subscriber:

- `"active"` and `"all"` refetches exclude unobserved queries.
- Named standby queries are refetched.
- From 4.0.11, a `skipToken` query is excluded until first executed with
  variables.

`useLazyQuery` is execute-driven. Changing hook options does not start a
request. Put `variables` and `context` on the execute call, keep other options
on the hook, and never execute during render or SSR. Unmounting or a newer
execution aborts the current promise with `AbortError`; call `.retain()` only
when the request must finish.

```ts
const [execute] = useLazyQuery(QUERY, { fetchPolicy: "no-cache" });
const result = await execute({ variables, context }).retain();
```

### Observable and method changes

The client-v4-migration replaces `zen-observable` with RxJS.
`ObservableQuery` is a `Subscribable`, so wrap it with `from()` for APIs that
need an RxJS `Observable`, and replace old instance operators with `pipe`
operators. The direct conversion is `from(observableQuery)`.

Client 4.0.0 removes `ObservableQuery.setOptions()` in favor of `reobserve()`
and replaces `ObservableQuery.result()` with RxJS conversion:

```ts
await observable.reobserve(newOptions);
const result = await firstValueFrom(from(observable));
```

### Refetch events

Client 4.2.0 makes automatic event-driven refetching opt-in through
`RefetchEventManager`. Built-in `windowFocusSource` and `onlineSource` events
refetch active queries by default after their sources are registered.

`refetchOn` accepts a boolean, a per-event map, or a predicate receiving the
source and payload. Per-query maps merge with
`defaultOptions.watchQuery.refetchOn`; locally omitted events keep following a
default boolean or predicate.

```ts
const refetchEventManager = new RefetchEventManager({
  sources: {
    windowFocus: windowFocusSource,
    online: onlineSource,
  },
});

const client = new ApolloClient({
  cache,
  link,
  refetchEventManager,
  defaultOptions: { watchQuery: { refetchOn: false } },
});
```

Custom events augment `RefetchEvents` with a payload type and provide an
Observable source, or register `true` for an imperative-only event and call
`emit`. Per-event `handlers` can replace normal active-query refetching and
receive `matchesRefetchOn`; `defaultHandler` or `setDefaultEventHandler`
changes the fallback. A handler returns `RefetchQueriesResult` or `void`.

Preloaded query references also honor declared watch-query defaults in 4.2.0.
For example, declared `errorPolicy: "all"` produces a reference whose state
union is
`PreloadedQueryRef<TData, TVariables, "complete" | "streaming" | "empty">`.

## Mutations and error handling

### Mutation completion and result synchronization

Since 3.13.0, an exception thrown by a `useMutation` `onCompleted` callback
rejects the mutation promise and is not forwarded to that mutation's
`onError`. Catch it from the returned promise.

The `ignoreResults` mutation option is deprecated since 3.13.0. To avoid
synchronizing a mutation result into component state, remove
`useMutation.ignoreResults` and call the client directly:

```ts
const client = useApolloClient();
await client.mutate({ mutation, variables });
```

Client 4.1.0 lets the mutate call's `context` be a callback over hook-level
context, so call-specific values can extend rather than replace defaults.

```ts
const [mutate] = useMutation(SAVE_ITEM, {
  context: { trace: true },
});
await mutate({
  context: hookContext => ({ ...hookContext, urgent: true }),
});
```

### Unified errors and policies

The client-v4-migration removes `ApolloError` and exposes only `result.error`.
Classify grouped failures with guards such as
`CombinedGraphQLErrors.is(error)` and `CombinedProtocolErrors.is(error)`.
Network errors pass through unchanged; unusual thrown values become
`UnconventionalError`. `ServerError` and `ServerParseError` are classes with
`.is` guards, and `ServerError.bodyText` replaces the parsed `result`.

All error sources follow `errorPolicy`:

- `none` rejects.
- `all` resolves with `result.error`.
- `ignore` resolves without the error.

`ObservableQuery` delivers failures in `next` results instead of terminating
through the observer's `error` callback. Subscription GraphQL errors behave
the same way, although an unrecoverable network failure can terminate a
subscription.

Client 4.0.0 gives `ErrorLink` callbacks one `error` property rather than
separate GraphQL, network, and protocol error properties. Use combined-error
guards, and use `LinkError.is(error)` for link-chain failures.

Client 4.2.0 makes `client.query` data non-nullable when the effective policy is
`"none"`. `ApolloClient.MutateResult<TData, TErrorPolicy>` maps policies as
follows:

- `"none"`: `data: TData`, no error.
- `"all"`: possibly undefined data and optional error.
- `"ignore"`: possibly undefined data and no error.

`client.mutate` and `useMutation` use the declared mutation default unless the
call overrides it; `useMutation.Result.error` is `undefined` for `"ignore"`.

### Pagination

In 4.0.0, `fetchMore` has its own `errorPolicy: "none"` default. Pass another
policy when partial pagination data is acceptable. Without a replacement query,
variables are shallow-merged; with a replacement query, supplied variables are
used as-is. `fetchMore` throws for a `cache-only` query.

## Type system and defaults

### Namespaced types and context

The client-v4-migration moves types under their APIs, such as
`ApolloLink.Result`, `ApolloClient.Options`, `ObservableQuery.Result`, and
`useQuery.Options`. It removes the `TContext` generic; augment
`DefaultContext` instead. The `ApolloClient` cache-shape and `ApolloCache`
serialization generics are also removed.

```ts
declare module "@apollo/client" {
  interface DefaultContext {
    requestId?: string;
  }
}
```

### Default-aware signatures

From 4.2.0, when `defaultOptions` sets `errorPolicy` or `returnPartialData`,
declare the values under `ApolloClient.DeclareDefaultOptions` so result types
reflect them. Required declarations select modern, document-inferred
signatures, which do not accept manual result generics.
The watch-query declaration path is `DeclareDefaultOptions.WatchQuery`.

When multiple clients have conflicting defaults, declare a narrow union.
Optional properties also include the runtime default—`"none"` for
`errorPolicy`—in inferred results.

Client 4.2.0 also permits an explicit `TypeOverrides.signatureStyle`:
`"modern"` selects document-inferred, default-aware signatures without a
required default declaration; `"classic"` temporarily preserves signatures
that accept generics, but declared defaults then stop affecting return types.

```ts
declare module "@apollo/client" {
  interface TypeOverrides {
    signatureStyle: "modern";
  }
  namespace ApolloClient {
    namespace DeclareDefaultOptions {
      interface WatchQuery {
        errorPolicy?: "all" | "ignore";
      }
    }
  }
}
```

## Incremental delivery and subscriptions

For the client-v4-migration, `@defer` requires an incremental handler. Initial
Client 4 support uses `Defer20220824Handler`. Custom-link incremental result
types and GraphQL Code Generator masking types are enabled through
`TypeOverrides` declaration merging.

```ts
new ApolloClient({
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

Client 4.1.0 adds `GraphQL17Alpha9Handler` for the GraphQL.js 17 alpha 9
format. Keep `Defer20220824Handler` for the older format. A mismatch can yield
errors or malformed data.

Also in 4.1.0, `Defer20220824Handler` and `GraphQL17Alpha2Handler` support
`@stream`; using `@stream` without a handler throws. The older defer handler
lacks stream metadata and truncates the existing array on the first chunk,
whereas the stream-aware default merge truncates on the final chunk.

Multipart query deduplication remains active until the final response chunk
from 3.13.0.

In the client-v4-migration, subscriptions are deduplicated by default. A
subscriber joining an existing connection does not receive the initial server
value. Opt out per request with
`context: { queryDeduplication: false }`.

Client 4.0.0 makes `client.subscribe()` lazy: the connection starts on
subscription. The returned observable exposes `restart()`, which replaces its
current link connection and recreates the request.

## Links and HTTP transport

The client-v4-migration favors class links such as `HttpLink`,
`SetContextLink`, `ErrorLink`, `PersistedQueryLink`, and
`RemoveTypenameFromVariablesLink`. `SetContextLink` reverses the old callback
arguments to `(previousContext, operation)`. Use `ApolloLink` static
composition; static `concat` is itself deprecated in favor of `from`, and
`from`, `concat`, and `split` require `ApolloLink` instances rather than bare
handlers.

For custom links:

- `operation.getContext()` is frozen; mutate through `setContext()`.
- Use `operation.client.cache` instead of context `cache` and `getCacheKey`.
- Use `operation.operationType` instead of parsing the document.
- Pass `{ client }` as the third argument to `execute(link, request, { client })`.

HTTP links advertise
`application/graphql-response+json,application/json;q=0.9` and interpret status
codes according to media type. A non-200 `application/json` response is a
`ServerError`; make mocks use production content types.

Client 4.0.0 enables enhanced client-awareness extensions by default on
`HttpLink` and `BatchHttpLink`: `includeExtensions` defaults to `true` and
`extensions.clientLibrary` carries library name and version. Disable it with
`enhancedClientAwareness: { transport: false }` when needed. Version 4.1.0 also
supports a header transport.

## Cache and local state

The 3.14.0 cache policy fix prevents field policies declared on a subtype from
overwriting or merging into the supertype's field policy.

The client-v4-migration makes local state opt-in:

```ts
new ApolloClient({
  cache,
  link,
  localState: new LocalState({ resolvers }),
});
```

Resolver context becomes `{ requestContext, client, phase }`. Resolution turns
`undefined` into `null` with a warning; non-scalar objects require
`__typename`; thrown resolver exceptions become GraphQL errors; and `@export`
requires a matching variable declaration with non-null values for required
variables.

Custom caches must implement `fragmentMatches` in Client 4.
`InMemoryCache` already does; `LocalState` throws when a custom cache omits it.

Client 4.1.0 adds these cache behaviors:

- `cache.write`, `cache.writeQuery`, and `client.writeQuery` accept
  `extensions`, and operation extensions reach field-policy `merge`.
- Array field reads preserve explicit `undefined` entries, allowing partial
  arrays to trigger a network fetch.
- A cache `read` for `@client` receives `existing: undefined`, so default
  parameters work.
- A custom `ApolloCache.resolvesClientField()` can return `true` to tell
  `LocalState` it supplies an otherwise unresolved local field; otherwise the
  client warns and returns `null`.

The fragment APIs `useFragment`, `useSuspenseFragment`, and
`client.watchFragment` accept arrays in `from` and return index-aligned arrays
in 4.1.0. Fragment watches accept `from: null` and emit complete null data.
The emitted shape is
`{ data: null, dataState: "complete", complete: true }`.
`readFragment`, `watchFragment`, and `updateFragment` expose `from`.

A query whose every field is skipped returns `{}` rather than `null` in 4.1.0,
which also prevents `useSuspenseQuery` from suspending forever.

## SSR and testing

The 3.14.0 preloaded-query API replaces `queryRef.toPromise()` with
`preloadQuery.toPromise(queryRef)`.

Client 4.0.0 replaces `getDataFromTree`, `getMarkupFromTree`, and
`renderToStringWithData` with `prerenderStatic`, including Suspense hooks and
React 19 static rendering. In 4.1.0 it returns the `renderFunction` value and
reports `aborted` correctly for React 19.2 `resumeAndPrerender` flows.

The client-v4-migration gives `MockLink` mocks a random 20–50 ms delay unless
configured, exercising loading states. Set
`MockLink.defaultOptions = { delay: 0 }` only when immediate global behavior is
required.

Client 4.0.0 removes `variableMatcher`; set `request.variables` to a predicate
when one mock should match multiple variable values.
