# Planning, Execution, Validation, and Demand Control

## Query-planning cancellation and resources

Router 2.4.0 adds
`supergraph.query_planning.experimental_cooperative_cancellation`. Use
`measure` to observe a timeout without cancelling or `enforce` to cancel
overlong planning. `apollo.router.query_planning.plan.duration` and the
`query_planning` span expose an `outcome`.

```yaml
supergraph:
  query_planning:
    experimental_cooperative_cancellation:
      enabled: true
      mode: measure
      timeout: 1s
```

Router 2.11.0 adds `memory_limit`. In `enforce` mode it cancels and errors; in
`measure` it records and lets planning finish. Memory and time limits race.
Memory accounting is available only on Unix with `global-allocator` enabled
and `dhat-heap` disabled. The full setting is
`experimental_cooperative_cancellation.memory_limit`.

`apollo.router.request.memory` measures allocation across a full request and
`apollo.router.query_planner.memory` covers planning compute jobs. Both expose
`allocation.type` and `context` and have the same build constraints.

Warm-up parsing and planning after hot reload run below live-request priority
from 2.4.0. Compute duration, queue wait, execution duration, and active-job
instruments label the work `query_parsing_warmup` or
`query_planning_warmup` in `job.type`.

## Planner and schema compatibility

The native planner rejects a supergraph schema containing an unknown `@link`
specification with purpose `EXECUTION` or `SECURITY` from 2.4.0. Remove or
correct the unknown link.

Router 2.6.0 supports labeled `@override` on fields and types implementing
interfaces in the same subgraph. Earlier native planning failed those
operations.

Router 2.16.0 preserves dependencies that supply entity keys, `__typename`, or
other values needed by a deferred query-plan block. This prevents fields in
the deferred chunk becoming null or disappearing.

## Variable and query validation

Router 2.12.0 can replace all query-validation failures with one generic
`invalid query` error carrying `UNKNOWN_ERROR`:
the setting is `supergraph.redact_query_validation_errors`.

```yaml
supergraph:
  redact_query_validation_errors: true
```

The same release validates fields inside input-object variables, including
rejecting unknown fields. Set `supergraph.strict_variable_validation: measure`
only when temporarily retaining non-enforcing behavior while observing
incompatibilities.

Router 2.12.0 also reports parser complexity through
`apollo.router.operations.recursion` and
`apollo.router.operations.lexical_tokens`.

## Result coercion and malformed responses

Router 2.8.0 can enable result-coercion errors:

```yaml
supergraph:
  enable_result_coercion_errors: true
```

Subgraph values that conflict with the schema and operation are reformatted and
nullified. With errors enabled, the Router adds
`RESPONSE_VALIDATION_FAILED` containing the path and reason, so clients can see
failures that were previously silent.

Router 2.16.0 extends this to a requested field missing from the merged
response. It emits one `RESPONSE_VALIDATION_FAILED` and one value-completion
entry at the source, while suppressing redundant errors along the null-bubble
path.

Router 2.5.0 replaces the nonstandard `"@"` path used for a malformed subgraph
array response with one error per concrete array index. Consumers must tolerate
more errors and expanded paths.

Router 2.13.0 attaches an entity-resolution error with no identifiable target
to the immediate parent rather than every expected entity, avoiding
multiplicative fan-out.

For subgraphs that return wrong entity error paths, 2.13.0 adds
`experimental_hoist_orphan_errors`. It places orphan errors at the nearest
non-array ancestor. Named settings override `all`; this reduces but does not
hard-cap error counts.

## Demand-control sizing

### Per-subgraph static estimates

Router 2.12.0 lets static estimated demand control define defaults and named
subgraph overrides. It sums every fetch to each subgraph across the plan.
Exceeding a subgraph limit skips only that subgraph's calls and composes those
values as null; other fetches continue.

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
```

### List sizing

Router 2.12.0:

- Processes every `@listSize` on a field and uses the largest `assumedSize`.
  At that release this awaits Federation composition support for repeatable
  directives.
- Understands array sizing values, nested input paths used to resolve sizes,
  and nested field paths in `sizedFields`.

Router 2.14.0 uses the length of a list-typed slicing argument as the cost
multiplier, whether the argument is inline or a variable.

### Actual cost

Router 2.12.0 changes
`demand_control.strategy.static_estimated.actual_cost_mode` to default to
`by_subgraph`, summing response cost from every subgraph fetch rather than only
the final response shape. Set `response_shape` to restore the earlier
calculation.

## Error and selector semantics

Router 2.4.0 makes `on_graphql_error` consistently boolean: it returns `false`
instead of absent when no GraphQL error exists, matches
`subgraph_on_graphql_error`, and works at supergraph as well as Router stage.

Value-completion failures are not present in the GraphQL errors array. From
2.1.0 they nevertheless increment `apollo.router.graphql.error` and
`apollo.router.operations.error` with
`code="RESPONSE_VALIDATION_FAILED"`.

Connector and demand-control errors include original message and path in
GraphOS traces from 2.3.0, in addition to their error code.
