# Subscriptions and Streaming Transport

## Client subscription behavior

During the client-v4-migration, Client subscriptions are deduplicated by
default. A subscriber joining an existing connection does not receive the
connection's initial server value. Disable deduplication per subscription when
that matters:

```ts
client.subscribe({
  query: SUBSCRIPTION,
  context: { queryDeduplication: false },
});
```

Apollo Client 4.0.0 makes `client.subscribe()` lazy. Creating the observable
does not open a connection; subscribing does. The observable's `restart()`
method closes its current link connection and recreates the request.

GraphQL subscription errors follow `errorPolicy` in Client 4 and normally
arrive through observer `next`, not `error`. An unrecoverable network failure
can still terminate the subscription.

## Router subscription deduplication

Router 2.3.0 adds `subscription.deduplication.ignored_headers`. Header
differences listed there no longer keep otherwise identical subgraph
subscriptions separate.

```yaml
subscription:
  enabled: true
  deduplication:
    enabled: true
    ignored_headers:
      - x-transaction-id
      - user-agent
```

Router 2.15.0 clarifies that decoded JWT claims participate in subscription
identity independently of forwarded headers. `ignored_headers` cannot make
authenticated requests share a subgraph connection.

The same release adds per-subgraph defaults and overrides and
`ignore_auth_context: true`. Use that only for non-personalized event streams:

```yaml
subscription:
  deduplication:
    all:
      enabled: true
    subgraphs:
      stocks:
        ignore_auth_context: true
```

## WebSocket protocol behavior

Router 2.4.0 makes subscription responses emitted during WebSocket handshake
satisfy GraphQL response validation, including with a coprocessor. Earlier
handshake responses could omit required `data`.

Router 2.7.0 accepts a `graphql-transport-ws` subgraph `connection_error` with a
payload but no `id` and propagates its underlying errors.

Router 2.9.0 delegates ping handling to the WebSocket implementation, avoiding
duplicate pong frames before acknowledgement. Its
`apollo.router.operations.subscriptions.events` counter increments for each
subscription event but excludes ping, pong, and close.

Router 2.11.0 injects trace propagation headers into the initial HTTP upgrade
to a subgraph. Individual messages on the established WebSocket cannot receive
new propagation headers.

## Multipart HTTP subscriptions and deferred results

Apollo Client query deduplication remains active through the final multipart
chunk from 3.13.0. Client 4.0.0 marks unfinished deferred data as
`dataState: "streaming"` and retains `loading: true` with
`NetworkStatus.streaming`.

For multipart HTTP subscriptions, Router 2.6.0 keeps a GraphQL-level error
followed immediately by stream completion at the GraphQL layer rather than
misclassifying it as a fatal transport error.

Router 2.13.0 can run multipart subscriptions behind AWS REST API Gateway after
that gateway gained response streaming. Configure the gateway response
transfer mode for streaming.

Incremental delivery protocols are not interchangeable. Align Client handler,
Server executor, GraphQL.js alpha, `Accept` header, gateways, mocks, and
proxies. See the Client and Server references for protocol-specific values.

## Lifetime and availability

Self-hosted subscriptions are available across Free, Developer, Standard, and
Enterprise GraphOS plans from Router 2.11.0. The self-hosted Router must still
connect to GraphOS using an API key and graph reference because subscriptions
are licensed.

Router 2.15.0 adds `subscription.max_lifetime`. On expiry, the Router closes
the stream with terminal error `SUBSCRIPTION_MAX_LIFETIME_EXCEEDED`. Unset
preserves unlimited lifetime.

```yaml
subscription:
  enabled: true
  max_lifetime: 10m
```

## Termination observability

Router 2.14.0 dynamically adds:

- `apollo.subscription.end_reason`: `server_close`, `subgraph_error`,
  `heartbeat_delivery_failed`, `client_disconnect`, `schema_reload`, or
  `config_reload`.
- `apollo.defer.end_reason`: `completed` or `client_disconnect`.

It also adds counters for client termination, rejected subscriptions, and
subgraph WebSocket closure:

- `apollo.router.operations.subscriptions.terminated.client`
- `apollo.router.operations.subscriptions.rejected`
- `apollo.router.operations.subscriptions.terminated.subgraph`

Router 2.4.0 labels `apollo.router.opened.subscriptions` with
`graphql.operation.name`, enabling per-operation open-stream monitoring.
