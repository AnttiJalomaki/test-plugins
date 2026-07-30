# Operations and Observability

## Server startup and scheduler limits

Nomad 1.10.1 adds `server.start_timeout` for setup and startup work such as
keyring decryption:

```hcl
server {
  start_timeout = "1m"
}
```

It defaults to `30s`. If startup work does not finish before the timeout, the
server logs the errors and exits.

`num_schedulers` must be between zero and the machine's available CPU count.
The timeout is from batch `1.10-upgrade`; the scheduler validation is from
batch `1.10.0`.

## Allocation metrics opt-in

Starting in 1.10.2, clients collect and publish allocation metrics only when
explicitly enabled:

```hcl
telemetry {
  publish_allocation_metrics = true
}
```

An unset or false `telemetry.publish_allocation_metrics` disables both
collection and publication. This behavior is from batch `1.10-upgrade`.

## Evaluation metric label changes

For dispatch and periodic jobs, the `job` label contains the parent job ID on:

- `nomad.nomad.broker.wait_time`
- `nomad.nomad.broker.process_time`
- `nomad.nomad.broker.response_time`
- `nomad.nomad.broker.eval_waiting`

The `nomad.nomad.broker.eval_waiting` metric no longer has an `eval_id` label.
Update queries and alerts that rely on the old labels. These changes are from
batch `1.11-upgrade`.

## CLI links to the web UI

Common CLI commands show web UI URL hints by default and accept `-ui` to open
the generated link. Disable hints on the server:

```hcl
ui {
  show_cli_hints = false
}
```

For one CLI environment, set `NOMAD_CLI_SHOW_HINTS=0` or
`NOMAD_CLI_SHOW_HINTS=false`. This CLI behavior is from batch `1.10.0`.

## Node API resource fields

The Go API `Node.Resources` and `Node.Reserved` fields, and their Read Node API
counterparts, are deprecated and never populated. Use `Node.NodeResources` and
`Node.ReservedResources`. This API migration is from batch `1.11-upgrade`.

## Event stream additions

CSI volume and plugin events are available in the event stream as of batch
`1.10.0`. Nomad variables emit events as of batch `2.0.0`, allowing consumers
to observe variable activity without polling.

## Raft log-store inspection

After the WAL capability introduced in batch `2.0.0`, `/v1/agent/self`
includes Raft log-store details and the WAL backend exposes log-store metrics.
See [upgrades-and-raft.md](upgrades-and-raft.md) for the one-way migration
procedure and snapshot requirement.

## Enterprise reporting and platform support

Nomad Enterprise 1.10.6 adds detailed product-usage information to automated
license-utilization reporting (batch `1.10-upgrade`).

Nomad Enterprise 2.0 supports Linux on the `ppc64le` CPU architecture (batch
`2.0.0`).
