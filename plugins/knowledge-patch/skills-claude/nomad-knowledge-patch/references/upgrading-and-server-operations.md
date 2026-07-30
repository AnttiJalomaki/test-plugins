# Upgrading and Server Operations

Use this reference for cluster sequencing, server lifecycle, Raft changes,
joins, server limits, and licensing.

## Upgrade sequencing and compatibility

Nomad aims for backward compatibility across at least two point releases, for
example a 1.7.x agent with a 1.5.x agent, but does not support downgrades. Do
not use new features until every relevant node is upgraded. Downgrading a
client requires draining its allocations and removing its data directory;
safely downgrading servers requires reprovisioning the cluster. (Source batch
`upgrade-procedure`.)

Upgrade servers one at a time before clients. A client restart that exceeds
`heartbeat_grace`, which defaults to `10s`, can cause every allocation on that
node to be rescheduled. Drain old clients when replacing them rather than
upgrading them in place. In a federated deployment, new features are not
guaranteed until every agent in the region and the authoritative region's
servers are upgraded.

For an in-place server upgrade, use a shutdown signal that does not trigger
the configured `leave_on_terminate` or `leave_on_interrupt`. For example, if
`leave_on_terminate` is enabled, use `SIGINT` rather than `SIGTERM`. After each
server rejoins:

1. Check membership with `nomad server members`.
2. Compare its `nomad agent-info` `last_log_index` with the other servers.
3. Proceed only after replication is current.

When replacing a server, stop it and confirm it is `left`; otherwise remove it
with:

```shell
nomad server force-leave <server-id>
```

Before upgrading servers to Nomad Enterprise 1.6.0 or later, validate the
license using `nomad license inspect` from the target Nomad binary.

## Raft protocol 3

Raft protocol 3 requires Nomad 0.8.0 or later on every server. Once all servers
use protocol 3, a server on an older protocol cannot join: quorum membership
identifies servers by node ID instead of IP address, and the outage-recovery
`peers.json` format changes.

For a cluster with at least three servers, stop and force-leave one server at
a time, restart it with protocol 3, then verify `RaftProtocol` using
`nomad operator raft list-peers` and replication using `nomad agent-info`.
Leave the leader until last. Set `raft_protocol = 3` explicitly only when
upgrading to a Nomad version earlier than 1.3.0.

```hcl
server {
  raft_protocol = 3
}
```

A single server cannot elect itself after an in-place protocol 3 restart
unless a new-format `server/raft/peers.json` is written before restarting.
Build it from the configured data directory, current leader address, and the
server node ID:

```shell
NOMAD_DATA_DIR=$(nomad agent-info -json | jq -r '.config.DataDir')
NOMAD_ADDR=$(nomad agent-info -json | jq -r '.stats.nomad.leader_addr')
NODE_ID=$(cat "$NOMAD_DATA_DIR/server/node-id")

cat >"$NOMAD_DATA_DIR/server/raft/peers.json" <<EOF
[
  {
    "id": "$NODE_ID",
    "address": "$NOMAD_ADDR",
    "non_voter": false
  }
]
EOF
```

## Raft log store and WAL migration

Nomad 2.0.0 deprecates the `raft_boltdb` server parameter; configure
`raft_logstore` instead (source batch `2.0-upgrade`). The following command
migrates the Raft log store from BoltDB to WAL:

```shell
nomad operator raft migrate-backend
```

This migration cannot be reversed in place. Returning to BoltDB requires
restoring a snapshot taken before migration. The `/v1/agent/self` response
includes Raft log store details, and the WAL backend exposes Raft log store
metrics (source batch `2.0.0`).

## Server joins

Unauthenticated use of `nomad server join` and the Join Agent API is deprecated
in 2.0.4. Nomad 2.1.0 requires a token carrying `agent:write`. Run a join
against the region leader when adding a node or against the authoritative
region when federating a region. For a new cluster, prefer `server_join` with
gossip encryption and mTLS.

The deprecated `server.retry_join`, `server.retry_interval`,
`server.retry_max`, and `server.start_join` parameters are removed in 2.1.0.
Migrate them to `server.server_join` before upgrading.

## Startup, capacity, and support

Nomad 1.10.1 adds `server.start_timeout`. It defaults to `30s` and bounds setup
and startup work such as keyring decryption. If work does not finish in time,
the server logs the errors and exits (source batch `1.10-upgrade`).

```hcl
server {
  start_timeout = "1m"
}
```

`num_schedulers` must be between zero and the machine's available CPU count.
Revalidate server configuration after changing CPU allocation or scheduler
count (source batch `1.10.0`). Nomad releases before 1.10.0 are outside the
support floor described by this upgrade guidance.

## Enterprise licensing

Nomad Enterprise 1.10.6 adds detailed product-usage information to automated
license utilization reporting. Nomad Enterprise 2.0 also includes license and
configuration changes for IBM Passport Advantage Online (PAO). Review
reporting and license configuration as part of an Enterprise upgrade.
