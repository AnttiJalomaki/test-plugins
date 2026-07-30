# CLI, API, and Observability

Use this reference when updating CLI automation, Go API consumers, event
subscribers, dashboards, metrics queries, or placement diagnostics.

## CLI links and allocation group selection

Common CLI commands display web UI URL hints by default and accept `-ui` to
open the generated link. Disable hints server-side:

```hcl
ui {
  show_cli_hints = false
}
```

For one CLI environment, set `NOMAD_CLI_SHOW_HINTS=0` or `false`. Account for
the hint output in scripts that parse human-readable command output (source
batch `1.10.0`).

`nomad alloc exec`, `nomad alloc logs`, and `nomad alloc fs` accept `-group`.
Use it when task names or allocation context are ambiguous across task groups.

`nomad volume status` shows volume capabilities. `nomad volume delete` accepts
a volume ID prefix and a wildcard namespace; resolve broad selectors before
performing a deletion.

## Evaluation and placement diagnostics

`nomad eval status` shows related evaluations, placed allocations, plan
annotations, failed placements, and preemptions. More fields are visible
without `-verbose`, so parsers of display output should not assume the older
terse shape.

Reconciler annotations describe the intended plan before node-feasibility
checks. `nomad alloc status -verbose` adds evaluated and rejected node counts
and node scores. In the Go API, `Evaluations.Info` populates `RelatedEvals`
(source batch `1.11.0`).

## Go API migrations

The Go API fields `Node.Resources` and `Node.Reserved`, and the corresponding
Read Node API fields, are deprecated and never populated. Use
`Node.NodeResources` and `Node.ReservedResources` (source batch
`1.11-upgrade`).

Quota clients must replace `QuotaSpec.VariablesLimit` with
`QuotaSpec.RegionLimit.Storage.Variables`. `QuotaSpec.RegionLimit` uses
`QuotaResources` instead of `Resources`.

## Allocation metrics opt-in

Starting in 1.10.2, clients do not collect or publish allocation metrics when
`telemetry.publish_allocation_metrics` is unset or false. Enable it explicitly
on every client that must continue exporting those metrics:

```hcl
telemetry {
  publish_allocation_metrics = true
}
```

Do not diagnose absent allocation series solely as a scraper failure; inspect
the client setting first (source batch `1.10-upgrade`).

## Evaluation broker metric labels

For dispatch and periodic jobs, the `job` label contains the parent job ID on:

- `nomad.nomad.broker.wait_time`
- `nomad.nomad.broker.process_time`
- `nomad.nomad.broker.response_time`
- `nomad.nomad.broker.eval_waiting`

The `nomad.nomad.broker.eval_waiting` metric no longer has an `eval_id` label.
Update queries, recording rules, alerts, and dashboard grouping that depended
on the child job ID or the removed label.

## Event stream additions

CSI volume and plugin events are included in the event stream. Nomad variables
also emit events, so consumers can observe variable activity without polling
(source batch `2.0.0`). Design subscribers to tolerate event types they do not
consume and to preserve ordering and resumption behavior.

## Enterprise reporting

Automated Nomad Enterprise license utilization reporting includes detailed
product-usage information. Review outbound reporting expectations and
operational documentation when upgrading Enterprise servers.
