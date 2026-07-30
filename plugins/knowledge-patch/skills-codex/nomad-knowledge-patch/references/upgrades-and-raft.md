# Upgrades and Raft

## Upgrade and downgrade boundaries

Nomad aims to remain backward compatible across at least two point releases;
for example, 1.7.x can coexist with 1.5.x. This is an operating goal, not
downgrade support. Do not enable new features until every agent that needs them
has been upgraded. In a federated deployment, that includes every agent in the
region and the authoritative region's servers.

Downgrades are unsupported:

- To downgrade a client, drain its allocations and remove its data directory.
- To downgrade servers safely, reprovision the cluster.

These constraints come from the `upgrade-procedure` batch.

Versions before 1.10.0 are outside the support floor (batch `1.10.0`).

## Rolling upgrade order

Upgrade servers one at a time, then upgrade clients. A client that takes longer
than `heartbeat_grace` to restart can have all allocations rescheduled;
`heartbeat_grace` defaults to `10s`. Drain old clients when replacing them
instead of upgrading them in place.

For an in-place server upgrade, choose a shutdown signal that does not activate
the configured `leave_on_terminate` or `leave_on_interrupt`. For example, when
`leave_on_terminate` is enabled, use `SIGINT` rather than `SIGTERM`.

After every server rejoins:

1. Check cluster membership with `nomad server members`.
2. Compare its `nomad agent-info` `last_log_index` with the other servers.
3. Continue only after replication is current.

When replacing a server, stop it and confirm it reaches `left`. If necessary,
remove it explicitly:

```shell
nomad server force-leave <server-id>
```

## Enterprise preflight

Before upgrading servers to Nomad Enterprise 1.6.0 or later, validate the
license with the target binary:

```shell
nomad license inspect
```

Nomad 2.0.0 also introduces license and configuration changes for IBM Passport
Advantage Online (PAO), from batch `2.0-upgrade`.

## Raft protocol 3 on a multi-server cluster

Raft protocol 3 requires Nomad 0.8.0 or later on every server. Once all servers
use protocol 3, an older-protocol server cannot join: quorum membership
identifies servers by node ID rather than IP address. The outage-recovery
`peers.json` format changes as well.

For a cluster with at least three servers:

1. Stop and force-leave one server.
2. Restart it using protocol 3.
3. Verify `RaftProtocol` with `nomad operator raft list-peers`.
4. Verify replication with `nomad agent-info`.
5. Repeat one server at a time, leaving the leader until last.

Set `raft_protocol = 3` explicitly only when upgrading to a Nomad version
earlier than 1.3.0:

```hcl
server {
  raft_protocol = 3
}
```

## Raft protocol 3 on a single server

A single server cannot elect itself after an in-place protocol 3 restart unless
a new-format `server/raft/peers.json` exists before the restart. Build it from
the configured data directory, current leader address, and server node ID:

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

The protocol procedures are from the `upgrade-procedure` batch.

## Raft log-store migration

The `raft_boltdb` server parameter is deprecated as of Nomad 2.0.0. Configure
`raft_logstore` instead (batch `2.0-upgrade`).

Migrate an existing log store from BoltDB to WAL with:

```shell
nomad operator raft migrate-backend
```

Take a snapshot first. The migration cannot be reversed in place; returning to
BoltDB requires restoring a snapshot captured before migration. After the
migration, `/v1/agent/self` reports Raft log-store details, and the WAL backend
exports Raft log-store metrics. These migration behaviors are from batch
`2.0.0`.

## Server join migration

Manual `nomad server join` and Join Agent API calls without authentication are
deprecated in 2.0.4. Nomad 2.1.0 requires a token with `agent:write`.

- Run a node-addition command against the region leader.
- Run a region-federation command against the authoritative region.
- For a new cluster, prefer `server_join` with gossip encryption and mTLS.

The legacy `server.retry_join`, `server.retry_interval`, `server.retry_max`,
and `server.start_join` parameters are removed in 2.1.0. Migrate them to
`server.server_join` before that upgrade. Both changes are recorded in batch
`2.0-upgrade`.
