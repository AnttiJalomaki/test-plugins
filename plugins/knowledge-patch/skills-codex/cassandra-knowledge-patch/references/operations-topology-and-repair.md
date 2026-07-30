# Operations, topology, and repair

## Gossip and endpoint state

Delayed gossip shutdown messages cannot overwrite a restarted node's fresh
startup state (5.0.3). A rapid restart should no longer leave the node falsely
marked down because an old shutdown state arrived late.

Gossip-only and bootstrapping nodes receive DC, rack, and host-ID endpoint
state (5.0.3). Topology-aware observers can use those fields before the node is
normal.

Gossip converges when several fields of an endpoint state are updated
concurrently (5.0.5).

The default maximum interval used by the failure detector is calculated
correctly (5.0.7). Clusters that leave it at the default may see changed
failure-detection timing after an upgrade.

## Token metadata and node tools

Use `nodetool checktokenmetadata` to check whether `TokenMetadata` agrees with
gossip endpoint state (5.0.3):

```shell
nodetool checktokenmetadata
```

Tool initialization skips the DirectIO check (5.0.4), so management commands
do not require that storage capability check merely to start.

`nodetool` and related tools avoid sourcing `cassandra-env.sh` when it is not
needed (5.0.5). Tool wrappers must set their own required environment and must
not depend on unrelated side effects from that file.

`nodetool gcstats` reports direct-memory usage correctly (5.0.7). Reset alert
baselines that compensated for the previous value.

## JMX lifecycle and controls

The `StorageService` JMX MBean is registered while a node is bootstrapping
(5.0.5), allowing management clients to observe and control that phase.

`StorageService.dropPreparedStatements` is exposed over JMX (5.0.6), so an
operator can invalidate prepared statements through the management interface.

`StorageProxyMBean` exposes `NativeTransportMaxConcurrentConnectionsPerIp`
(5.0.6), making the per-IP native connection cap available to JMX inventory
and monitoring.

Handled exceptions no longer create heap dumps (5.0.7). Automation should not
wait for or collect a dump when Cassandra catches and handles the exception.

## Runtime and mixed-version behavior

Cassandra has full Java 17 runtime support (5.0.5).

Mixed-version Paxos no longer hangs on TTL commits or loops indefinitely
(5.0.4). During rolling upgrades, a TTL-bearing Paxos operation should
complete rather than needing the former operational workaround.

## Repair behavior

Long-running repairs are not automatically failed prematurely (5.0.4). Use
actual repair status and failure output rather than elapsed time alone to
classify a repair as failed.

## Built-in AutoRepair

An in-process automated repair scheduler is available (5.0.8), allowing
recurring repair to be scheduled without making external orchestration the
only option.

The scheduler supports a minimum repair-task duration setting (5.0.8), which
bounds scheduled work with a minimum run time.

`preview_repaired` is a supported AutoRepair type (5.0.8).

The scheduler stops when it sees two Cassandra major versions (5.0.8). Treat
AutoRepair as suspended during a mixed-major-version upgrade and arrange any
required repair coverage accordingly.

Full AutoRepair observes disk protection (5.0.8), preventing scheduled full
repair from proceeding without regard to disk-protection conditions.

Progress reporting includes expected and actual repair bytes plus expected
and actual keyspaces (5.0.8). Use both dimensions to assess scheduler progress
and scope.
