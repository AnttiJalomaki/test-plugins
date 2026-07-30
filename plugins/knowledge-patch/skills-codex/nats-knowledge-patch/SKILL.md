---
name: nats-knowledge-patch
description: NATS Server
version: 2.14.0
license: MIT
metadata:
  author: Nevaberry
---


# NATS Server Knowledge Patch

Use this skill when configuring, upgrading, operating, or writing integrations
for recent NATS Server and JetStream behavior. Check the deployed server version
and client-library capabilities before applying a feature, and trust the
project's configuration, tests, and observed server behavior when they differ.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/streams-and-publishing.md](references/streams-and-publishing.md) | Strict requests, per-message TTL, delete markers, buffering, atomic and fast batches, counters, schedules, mirrors, transforms |
| [references/consumers-and-sourcing.md](references/consumers-and-sourcing.md) | Priority groups, pausing, resets, reliable sources and mirrors, ACK and flow-control subjects |
| [references/networking-and-accounts.md](references/networking-and-accounts.md) | Routes, gateways, leafnodes, TLS, accounts, metadata, MQTT, names and reload behavior |
| [references/operations-and-recovery.md](references/operations-and-recovery.md) | Tracing, health, configuration digests, replication, storage, memory, upgrades and downgrades |

## Breaking defaults and upgrade traps

### Treat JetStream requests as strict

Unknown JSON fields are rejected by default. Fix request schemas instead of
depending on ignored fields. Use the temporary compatibility switch only while
migrating:

```text
jetstream {
  strict: false
}
```

### Budget stream ingest buffers

Core publishes entering JetStream are bounded per stream. The defaults are
10,000 messages and 128 MB; overflow can drop messages and return
`429 JSStreamTooManyRequests`.

```text
jetstream {
  max_buffered_msgs: 50000
  max_buffered_size: 256mib
}
```

Prefer JetStream publishes that wait for PubAcks. Size both limits for expected
bursts instead of assuming unlimited buffering.

### Update health and shutdown expectations

`js-server-only` does not test meta-leader health. Use `js-meta-only` when the
meta group is the intended signal. A graceful `SIGTERM` exits with status `0`.
Spaces in server, cluster, or gateway names are startup errors.

### Plan downgrade boundaries

- A first restart across a changed stream-state format rescans message blocks.
  Expect extra CPU use and delayed health, but not data loss.
- Downgrading assets that use newer features requires a compatible maintenance
  release that can place those assets safely offline.
- Reliable WorkQueue or Interest sourcing can revert to an ephemeral mode on
  downgrade; transition may interrupt sourcing.
- Remove the entire `feature_flags` block before moving to a server that does
  not recognize it.

Read [references/operations-and-recovery.md](references/operations-and-recovery.md)
before rolling upgrades, full-cluster recovery, or downgrade work.

## Choose the publishing mode deliberately

### Per-message expiration

Enable `AllowMsgTTL` on the stream, then publish `Nats-TTL` as integer seconds,
a Go duration, or `never`.

```go
StreamConfig{AllowMsgTTL: true}
```

`never` also bypasses stream `MaxAge`. Invalid or sub-second TTL values reject
and discard the publish, and the feature cannot later be disabled. Delete
markers, mirrors, sources, and clamping rules have additional constraints in
[references/streams-and-publishing.md](references/streams-and-publishing.md).

### Atomic batches

Use `AllowAtomicPublish` when the whole batch must commit or fail together.
Every message shares `Nats-Batch-Id` and uses a contiguous
`Nats-Batch-Sequence`; the final message carries `Nats-Batch-Commit: 1`.

```go
StreamConfig{AllowAtomicPublish: true}
```

Atomic publishing requires API level 2, synchronous persistence, and a
non-mirror stream. Only the final publish gets the normal PubAck.

### Fast batches

Use `AllowBatchPublish` for flow-controlled throughput when per-message
consistency checks are sufficient and atomic staging is unnecessary.

```go
StreamConfig{AllowBatchPublish: true}
```

Both atomic and fast batches can use a non-persisted end-of-batch commit
message.

### Counter streams

`AllowMsgCounter` turns each subject into an arbitrary-precision signed
counter. Each publish must include a signed `Nats-Incr`; the stored body and
PubAck contain the new total.

```bash
nats req counter.hits '' -J -H 'Nats-Incr:+2'
```

Counter mode is creation-only and has retention, mirror, discard, TTL, schedule,
and sourcing constraints. Consult the stream reference before designing
aggregation or reset behavior.

### Scheduled messages

`AllowMsgSchedules` supports one-time, cron, interval, and sampled-source
schedules. Keep each schedule on a unique subject and target another subject in
the same stream.

```bash
nats pub -J schedules.orders.once \
  -H 'Nats-Schedule: @at 2025-10-01T12:00:00Z' \
  -H 'Nats-Schedule-Target: orders' \
  -H 'Nats-Schedule-TTL: 5m' \
  'body'
```

Schedule-record TTL, generated-message TTL, and scheduled roll-up are distinct
controls.

## Configure consumers and sourcing

### Pick the grouped pull policy

Grouped pull consumers require explicit acknowledgements and one configured
priority group:

- `overflow` starts delivery when a pull's `min_pending` or `min_ack_pending`
  threshold is met.
- `pinned_client` selects a client and keeps other pulls as standbys. Persist
  `Nats-Pin-Id`; after a `423` mismatch, clear it and retry without an ID.
- `prioritized` gives a pull work sooner than overflow, but work can move back
  and forth between clients.

Only the pinned timeout is updatable. Do not expect an existing consumer to
enter or leave grouped mode or switch policies.

### Pause without losing liveness

Set `PauseUntil` at creation or through the pause API. Delivery resumes at the
deadline and heartbeats continue during the pause.

### Reset delivery state carefully

Publish an empty body or `{"seq": N}` to:

```text
$JS.API.CONSUMER.RESET.<STREAM>.<CONSUMER>
```

An external reset can make a non-ordered consumer's delivery sequence
non-monotonic. Arbitrary sequence resets apply only to supported start
policies and cannot bypass the configured lower bound.

### Secure reliable source consumers

WorkQueue and Interest mirrors and sources use visible, durable
`JS_MIRROR_<suffix>` or `JS_SRC_<suffix>` consumers. Pre-create one when its
lifecycle, permissions, filters, or replay policy must be explicit. Put
starting position and filters on that consumer rather than on `StreamSource`.

## Prepare ACK permissions for domain-aware subjects

Servers can parse both layouts and can emit the domain/account-aware form when
the restart-only flag is enabled:

```text
feature_flags {
  js_ack_fc_v2: true
}
```

Catch-all `$JS.ACK.>` and `$JS.FC.>` permissions cover both forms. Narrow rules
such as `$JS.ACK.<stream>.>` do not. Client parsers must accept both layouts
and publish the supplied reply subject unchanged.

## Operate networks and accounts

- Enable `handshake_first` when a leafnode must negotiate TLS before the NATS
  protocol handshake.
- Use `connect_backoff: true` on routes and gateways to trade faster retry for
  exponential reconnect delays from one to 30 seconds.
- Use `isolate_leafnode_interest` when leafnodes do not need east-west interest
  propagation.
- Leafnode remotes can be toggled with `disabled`; the whole remotes section
  can also be added or removed on reload.
- `cluster_traffic` can charge an asset's Raft replication to its owning
  account instead of the system account.

Review [references/networking-and-accounts.md](references/networking-and-accounts.md)
for TLS defaults, reload semantics, metadata, global-account events, and
Sparkplug behavior.

## Diagnose before changing state

- Compare `nats-server -t` output with `varz.config_digest` to detect an
  on-disk configuration that was not loaded.
- A stream filestore write error freezes that stream and fails health checks;
  other streams and core traffic continue. Restart the affected server to
  recover it.
- An overloaded Raft leader can step down. If a majority is overloaded,
  capacity must catch up before the cluster recovers.
- Distributed tracing uses `Nats-Trace-Dest`; add `Nats-Trace-Only: true` to
  collect hop events without subscriber delivery.
- Size `GOMEMLIMIT` from memory actually available to the process or container,
  not from historical RSS behavior.

The operations reference contains the full replication, leader-change,
recovery, encryption, and trace-header details.
