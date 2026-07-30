# Repair, Topology, and Coordination

## Built-in AutoRepair

### Scheduler model

Cassandra includes an in-process automated repair scheduler (since 5.0.8),
backporting CEP-37 behavior. Recurring repairs no longer have to be scheduled
exclusively by an external orchestrator, although external operational
coordination may still be needed around upgrades and disk protection.

### Task duration

AutoRepair has a minimum repair-task-duration setting (since 5.0.8). Use it
when scheduled work must respect a minimum run time rather than allowing every
task to end sooner.

### Repair types

`preview_repaired` is accepted as an AutoRepair repair type (since 5.0.8).
This lets the scheduler run preview-repaired work rather than limiting that
workflow to manually orchestrated repair.

### Mixed-major-version behavior

The scheduler stops when it detects two major Cassandra versions (since
5.0.8). During a mixed-major-version upgrade:

- do not assume scheduled repair remains active;
- distinguish an intentional scheduler stop from a hung repair;
- provide an upgrade-period repair plan if repair must continue.

### Disk protection

Full AutoRepair observes disk-protection safeguards (since 5.0.8). If a full
scheduled repair does not proceed, inspect disk-protection conditions before
retrying or treating the scheduler as defective.

### Progress reporting

AutoRepair reports expected versus actual repair bytes and expected versus
actual keyspaces (since 5.0.8). Compare both dimensions when deciding whether
work is progressing, incomplete, or divergent from its plan.

## Manual repair behavior

### Long-running repairs

Repairs that run for a long time are not automatically failed prematurely
(since 5.0.4). Do not infer failure solely from crossing an older implicit
duration expectation; inspect actual repair state.

### Repair flushes and SAI

When repair flushes a partial partition or row modification, SAI marks the
index non-empty (since 5.0.5). Queries after such a flush should not lose
visibility merely because index state remained incorrectly empty.

## Gossip and endpoint state

### Restart-safe state

A delayed gossip shutdown message cannot overwrite a restarted node's fresh
startup state (since 5.0.3). This prevents a successfully restarted node from
remaining falsely marked down because stale shutdown information arrived
late.

### Non-normal nodes

Gossip-only and bootstrapping nodes receive data-center, rack, and host-ID
endpoint state (since 5.0.3). Topology consumers can expect these fields even
before a node reaches normal state.

### Concurrent endpoint updates

Gossip converges when multiple endpoint-state fields are updated concurrently
(since 5.0.5). A multi-field update should not leave peers permanently split
across different endpoint-state combinations.

### Token metadata validation

`nodetool checktokenmetadata` checks whether `TokenMetadata` is synchronized
with gossip endpoint state (since 5.0.3):

```shell
nodetool checktokenmetadata
```

Run it when token ownership or topology views disagree, especially after
restart or bootstrap events.

## Hints and replica coordination

### Hint expiration

Hint TTL is calculated from the request start time rather than from the
timeout time (since 5.0.3). Expiry therefore reflects the age of the original
request.

### Schema mismatch

A schema mismatch no longer categorically blocks hint delivery (since 5.0.3).
Do not diagnose the mere presence of schema disagreement as proof that hints
cannot be delivered.

### Mixed-version Paxos

Mixed-version Paxos no longer hangs on TTL commits or enters an infinite loop
(since 5.0.4). Rolling-upgrade validation should include TTL-bearing Paxos
workloads because these operations have a specific corrected mixed-version
path.

### Parallel transfers

`MAX_PARALLEL_TRANSFERS` is honored correctly (since 5.0.5). Transfer planning
and throttling can rely on the configured limit being enforced.

## Batchlog endpoint placement

Batchlog endpoint selection accepts four strategies (since 5.0.3):

- `random_remote`
- `prefer_local`
- `dynamic_remote`
- `dynamic`

For example:

```yaml
batchlog_endpoint_strategy: dynamic_remote
```

Choose the strategy deliberately for the desired local or remote placement;
the setting is not limited to a single built-in selection behavior.

## Streaming compatibility

Zero-copy streaming is automatically disabled for legacy SSTables that use
the old Bloom-filter format (since 5.0.7). Those SSTables fall back to a
compatible streaming path. A fallback for such files is expected and should
not be treated as proof that zero-copy streaming is globally disabled.
