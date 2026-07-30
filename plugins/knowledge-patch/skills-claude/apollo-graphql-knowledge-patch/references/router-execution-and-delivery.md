# Apollo Router execution, demand control, and delivery

## Query planning

### Cooperative cancellation (`2.4.0`)

`supergraph.query_planning.experimental_cooperative_cancellation` can measure or
enforce a planning timeout. `measure` records an outcome without cancelling;
`enforce` cancels overlong work. The
`apollo.router.query_planning.plan.duration` metric and `query_planning` span carry
`outcome`.

```yaml
supergraph:
  query_planning:
    experimental_cooperative_cancellation:
      enabled: true
      mode: measure
      timeout: 1s
```

Cache warm-up parsing and planning run below live-request priority. Compute-job
duration, queue wait, execution time, and active-job instruments identify them with
`job.type=query_parsing_warmup` or `query_planning_warmup`.

### Memory ceiling (`2.11.0`)

`experimental_cooperative_cancellation.memory_limit` races with `timeout`.
`enforce` cancels and errors; `measure` records while allowing completion. It
requires Unix, `global-allocator` enabled, and `dhat-heap` disabled.

```yaml
supergraph:
  query_planning:
    experimental_cooperative_cancellation:
      enabled: true
      mode: enforce
      memory_limit: 50mb
      timeout: 5s
```

Request and planner allocation instruments have the same platform and allocator
constraints.

### Deferred dependencies (`2.16.0`)

Plan reduction preserves dependencies supplying entity keys, `__typename`, and
other values needed by a deferred block. This prevents deferred fields from
becoming null or disappearing.

## Demand control and operation complexity

### Per-subgraph ceilings (`2.12.0`)

Static estimated demand control supports `all` defaults and named-subgraph
overrides. It sums all fetches to a subgraph. If that subgraph exceeds its limit,
only its calls are skipped and composed as null; the rest of the operation
continues.

```yaml
demand_control:
  enabled: true
  mode: enforce
  strategy:
    static_estimated:
      max: 20
      list_size: 10
      subgraphs:
        all:
          max: 8
          list_size: 10
        subgraphs:
          products:
            max: 6
          reviews:
            list_size: 50
```

The cost engine understands array sizing values, nested input paths, and nested
`sizedFields`. It processes every repeatable `@listSize` and uses the largest
`assumedSize`; in the initial release this waits for composition support to place
repeatable directives in the supergraph.

`demand_control.strategy.static_estimated.actual_cost_mode` defaults to
`by_subgraph`, counting intermediate response work from every fetch. Set
`response_shape` for the former final-shape calculation.

```yaml
demand_control:
  strategy:
    static_estimated:
      actual_cost_mode: response_shape
```

From `2.14.0`, the length of a list-typed argument in
`@listSize(slicingArguments: [...])` is a cost multiplier whether supplied inline
or as a variable.

Parser complexity can be observed through operation recursion and lexical-token
metrics; see `router-observability.md`.

## Federation and planning correctness

The native planner rejects unknown `@link` specifications with purpose `EXECUTION`
or `SECURITY` (`2.4.0`), matching the legacy planner. Correct or remove the link.

Labeled progressive `@override` on fields or types implementing interfaces in the
same subgraph is supported from `2.6.0`; it previously caused planning failure.

Connector introduction preserves `@context` and `@fromContext` from Router 2.1.2.
Response caching recognizes interface objects as entities from `2.10.0`.

Root-type authorization and recursive Connector input behavior are covered in the
security and Connector references.

## Result coercion and response validation

### Subgraph mismatches (`2.8.0`)

The Router reformats and nullifies values that do not match the schema and query.
When result-coercion errors are enabled, it additionally emits
`RESPONSE_VALIDATION_FAILED` with path and reason:

```yaml
supergraph:
  enable_result_coercion_errors: true
```

From `2.16.0`, a requested field missing from the merged result produces one
`RESPONSE_VALIDATION_FAILED` and one source value-completion entry. Redundant
errors along its null-bubble path are suppressed.

Value-completion failures absent from the GraphQL `errors` array are counted as
`code="RESPONSE_VALIDATION_FAILED"` by `apollo.router.graphql.error` and
`apollo.router.operations.error` (`2.1.0`).

### Malformed errors and paths

From `2.5.0`, a malformed subgraph result affecting an array produces one concrete
path per index instead of the nonstandard `"@"` path. Clients must allow expanded
error counts and paths:

```json
{
  "errors": [
    { "path": ["topProducts", 0] },
    { "path": ["topProducts", 1] }
  ]
}
```

An unlocatable entity error attaches to its immediate parent from `2.13.0` rather
than every expected entity. For incorrect entity paths returned by a subgraph,
`experimental_hoist_orphan_errors` assigns errors to the nearest non-array
ancestor. Named settings override `all`; this reduces but does not cap counts.

```yaml
experimental_hoist_orphan_errors:
  subgraphs:
    my_subgraph:
      enabled: true
```

Errors for `_entities` fetches retain responsible subgraph or Connector service
attribution from `2.7.0`.

### Coprocessor errors

When a coprocessor returns an execution error with `data: null`, the Router
preserves the `data` member (`2.2.0`). Coprocessor response validation is enabled
by default from `2.5.0`.

### GraphQL error details

Connector and demand-control traces include their error codes from `2.1.0` and
include original message and path in GraphOS traces from `2.3.0`.

GraphQL error selectors consistently return a boolean from `2.4.0`:
`on_graphql_error` yields false rather than absence and works at the supergraph
stage as well as the router stage, matching `subgraph_on_graphql_error`.

Fine-grained redaction, telemetry selectors, and extended error events are covered
in the security and observability references.

## Incremental and subscription delivery

### Multipart correctness

Subscription payloads emitted during WebSocket handshake satisfy GraphQL response
validation from `2.4.0`, including with coprocessors.

For multipart HTTP subscriptions, a GraphQL-level error immediately followed by
termination remains a GraphQL error instead of being misclassified as a fatal
transport failure (`2.6.0`).

Router `2.7.0` propagates payload-bearing `graphql-transport-ws`
`connection_error` messages even when they have no `id`.

### Subscription event accounting

From `2.9.0`, `apollo.router.operations.subscriptions.events` increments for each
event but excludes ping, pong, and close. The Router relies on the WebSocket
implementation's ping handling, preventing duplicate pong replies before
acknowledgement.

### Termination reasons (`2.14.0`)

Router spans expose:

- `apollo.subscription.end_reason`: `server_close`, `subgraph_error`,
  `heartbeat_delivery_failed`, `client_disconnect`, `schema_reload`, or
  `config_reload`.
- `apollo.defer.end_reason`: `completed` or `client_disconnect`.

Counters distinguish subscriptions terminated by a client, rejected by a limit or
subgraph, and terminated by a subgraph WebSocket closure. Their exact names are
`apollo.router.operations.subscriptions.terminated.client`,
`apollo.router.operations.subscriptions.rejected`, and
`apollo.router.operations.subscriptions.terminated.subgraph`.

Maximum lifetime, deduplication, and deployment transports are detailed in
`router-caching-and-traffic.md`.
