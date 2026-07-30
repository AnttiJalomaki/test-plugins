# Operations and recovery

## Distributed message tracing

Since 2.11.0, publish with `Nats-Trace-Dest` set to an inbox to receive hop
events as the message enters or leaves servers, crosses connection types or
account boundaries, and undergoes subject mapping.

```text
Nats-Trace-Dest: trace.inbox
Nats-Trace-Only: true
```

`Nats-Trace-Only: true` propagates trace events without delivering the traced
message to subscribers. Since 2.14.0, distributed tracing preserves an
existing `traceparent` header instead of modifying it.

## Configuration digests

Since 2.11.0, the server's `-t` flag generates a hash of its configuration
file, and `varz.config_digest` exposes the hash of the running configuration.
Compare them to detect an on-disk change that has not been loaded.

## Health checks and graceful shutdown

Since 2.11.0, `js-server-only` no longer checks meta-leader health. Use the
`js-meta-only` check when meta-group health is the desired signal.

Graceful `SIGTERM` shutdown exits with status `0`.

## Replicated deletion and leader changes

Since 2.11.0, deletes in replicated Interest and WorkQueue streams are ordered
through Raft proposals. Account for the possible increase in replication
traffic.

A new leader waits to synchronize its Raft log before serving reads or writes.
Replicated consumers redeliver unacknowledged messages after a leader change.
Configured consumer start sequences are honored except on hidden source or
mirror consumers.

## JetStream asset API levels

The 2.11.0 line assigns 2.11.x JetStream API support level `1`. The server
advertises the level through `jsz`, `varz`, and `$JS.API.INFO`.

Server-managed assets record `_nats.ver`, `_nats.level`, and
`_nats.req.level`. Reconciliation tools must ignore these dynamic metadata
values. Level-dependent fields include a nonzero `PauseUntil` and the
message-TTL settings. Features that explicitly require level 2 must be
rejected or avoided when the asset level is lower.

## Windows TPM-backed filestore keys

Since 2.11.0 on Windows, JetStream filestore encryption keys can be protected
by the machine's TPM rather than only by storage available to an attacker with
physical access.

## Filestore memory behavior

Since 2.12.0, elastic filestore caches can be released under memory pressure,
so RSS may be higher or lower than older workload patterns suggest. Size
`GOMEMLIMIT` for memory actually available to the server, including container
reservations.

## Filestore write errors

Since 2.14.0, a filestore write error freezes only the affected stream,
produces a `write error` log entry, and fails health checks. Core traffic and
other streams continue. A replicated stream can fail over, but the affected
server must restart to recover.

## Raft overload containment

Since 2.14.0, Raft bounds memory and disk growth when proposals arrive faster
than they can commit. A lagging leader steps down for a healthier peer. If a
majority is overloaded, the cluster stays degraded until capacity catches up.

## Full-replica recovery

Since 2.12.0, recovery of a replicated in-memory stream after all but one
replica have restarted may require every replica to be available, rather than
only a quorum, while the server selects the state that preserves data.

## Upgrade and downgrade safeguards

### Stream-state rebuilds

On the first 2.11-to-2.10 restart, changed stream-state files are rebuilt by
rescanning message blocks. The rebuild does not lose data, but increases CPU
use and delays the node becoming healthy.

The first 2.12-to-2.11 restart also rescans message blocks, with the same
temporary CPU and health impact.

### Downgrading newer JetStream assets

When downgrading from 2.12, use 2.11.9 or newer so assets using 2.12-only
features are placed safely offline.

Downgrading reliable WorkQueue or Interest sources from 2.14 to 2.12 restores
their less reliable ephemeral mode and may interrupt sourcing during the
transition. `AckFlowControl` consumers remain offline until 2.14 is restored.

Before downgrading to a server without feature-flag support, remove the entire
`feature_flags` block. Unknown flag names are ignored but logged on servers
that support the block, and flags are restart-only rather than reloadable.
