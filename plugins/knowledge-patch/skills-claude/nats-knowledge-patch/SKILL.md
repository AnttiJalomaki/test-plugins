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
for NATS Server, especially JetStream. Prefer the project's server
configuration, client-library types, deployed version, and observed behavior
when they differ from generic examples.

## Reference index

| Reference | Topics |
| --- | --- |
| [jetstream-streams-and-publishing.md](references/jetstream-streams-and-publishing.md) | Strict requests, message TTL, delete markers, ingest limits, atomic and fast batches, counters, schedules, transforms, replication semantics |
| [consumers-mirrors-and-sources.md](references/consumers-mirrors-and-sources.md) | Priority groups, pausing, mirror promotion, reliable sourcing, consumer reset, ACK and flow-control subjects |
| [networking-security-and-observability.md](references/networking-security-and-observability.md) | Distributed tracing, routes, gateways, leafnodes, TLS, encryption, MQTT, metadata, system events |
| [operations-upgrades-and-recovery.md](references/operations-upgrades-and-recovery.md) | Configuration drift, health checks, API levels, shutdown, downgrade rebuilds, recovery, filestore and Raft overload |

## Breaking and compatibility changes

### Expect strict JetStream request validation

JetStream rejects JSON request fields it does not recognize. Fix request
schemas and client payloads instead of depending on ignored fields. During a
short migration window only, restore permissive handling with:

```text
jetstream {
  strict: false
}
```

### Budget for bounded stream ingest

Core NATS publishes entering JetStream are buffered per stream. The defaults
are 10,000 messages and 128 MB; overflow can drop messages and report
`429 JSStreamTooManyRequests`.

```text
jetstream {
  max_buffered_msgs: 50000
  max_buffered_size: 256mib
}
```

Prefer JetStream publishing that waits for PubAcks. Raise the limits only after
accounting for the resulting memory use.

### Treat names and health checks precisely

- Server, cluster, and gateway names containing spaces fail startup.
- `js-server-only` does not test meta-leader health. Use `js-meta-only` when
  the meta group is the signal you need.
- Graceful `SIGTERM` shutdown exits successfully with status `0`.

### Plan downgrade boundaries

The first restart across a stream-state format boundary can rescan message
blocks, increasing CPU use and delaying health without losing data. When
downgrading assets that use newer JetStream features, use a destination release
with the corresponding offline-asset safeguard. Reliable WorkQueue and
Interest sourcing also reverts to the older ephemeral mode when moving back
from the durable sourcing implementation.

### Prepare permissions for domain-aware ACK subjects

Servers accept both the original ACK/flow-control subject form and the
domain/account-aware form. Before enabling v2 emission, replace narrow rules
such as `$JS.ACK.<stream>.>` and `$JS.FC.<stream>.>` or extend them to cover
v2. Catch-all `$JS.ACK.>` and `$JS.FC.>` already cover both forms.

```text
feature_flags {
  js_ack_fc_v2: true
}
```

The feature-flag block is restart-only and must be removed before downgrading
to a server that does not recognize it.

## High-value JetStream publishing features

### Choose the right batch mode

Use `AllowAtomicPublish` for an all-or-nothing batch:

```go
StreamConfig{AllowAtomicPublish: true}
```

Atomic batches use one batch ID, contiguous sequences, and a commit on the
last message or an end-of-batch message. They stage intermediate messages and
produce a normal PubAck only when committed.

Use `AllowBatchPublish` for flow-controlled throughput with per-message
consistency checks but without atomic staging:

```go
StreamConfig{AllowBatchPublish: true}
```

Both modes can finish with an end-of-batch message that is not persisted.
Consult the publishing reference for atomic-mode limits and incompatible
settings.

### Apply per-message expiration deliberately

Enable `AllowMsgTTL` before publishing `Nats-TTL`:

```go
StreamConfig{AllowMsgTTL: true}
```

The header accepts integer seconds, Go durations, or `never`. Invalid or
sub-second TTLs reject the publish. `never` also exempts a message from stream
`MaxAge`, and the stream option cannot later be disabled.

Use `SubjectDeleteMarkerTTL` when consumers must observe that age-based
expiration removed a subject's last message. API delete and purge operations
do not create these markers.

### Create counters only for counter workloads

`AllowMsgCounter` turns every stream subject into an arbitrary-precision
signed counter. Publish a signed `Nats-Incr`; the stored body and PubAck carry
the resulting value.

```go
StreamConfig{AllowMsgCounter: true}
```

Counter mode is creation-only and excludes several ordinary stream features,
including per-message TTL, schedules, mirrors, and body-only publishes.

### Separate schedule lifetime from output lifetime

With `AllowMsgSchedules`, use `Nats-Schedule` for `@at`, six-field UTC cron, an
alias, or a Go-duration interval. `Nats-TTL` expires the schedule record;
`Nats-Schedule-TTL` becomes `Nats-TTL` on generated messages.

```go
StreamConfig{
    AllowMsgSchedules: true,
    AllowMsgTTL:       true,
}
```

Schedules can publish a supplied body, sample the latest message with
`Nats-Schedule-Source`, and apply a generated-message roll-up with
`Nats-Schedule-Rollup`.

## High-value consumer and sourcing features

### Select a priority-group policy

Grouped pull consumers require explicit acknowledgements and one configured
priority group:

```go
ConsumerConfig{
    PriorityGroups: []string{"jobs"},
    PriorityPolicy: "overflow",
    AckPolicy:      "explicit",
}
```

- `overflow` begins delivery when a pull's `min_pending` or `min_ack_pending`
  threshold is met.
- `pinned_client` selects one client and keeps other pulls as standbys. Persist
  the returned `Nats-Pin-Id`, retry without it after a `423`, and remember that
  already in-flight work is not strictly exclusive.
- `prioritized` admits a group sooner than overflow and can move work back and
  forth between clients.

Only the pin timeout is updatable; grouped mode and its policy cannot be
changed in place.

### Pause and reset consumers intentionally

Set `PauseUntil` or use the pause API to suspend delivery until a deadline.
Heartbeats continue and delivery resumes automatically.

Reset delivery state through:

```text
$JS.API.CONSUMER.RESET.<STREAM>.<CONSUMER>
```

An empty request clears pending/redelivery/delivered state while retaining the
stream ack floor. `{"seq": N}` advances the next stream sequence subject to
the consumer's delivery policy and configured start bound. A reset can make
delivery sequences non-monotonic to other clients.

### Control source consumers explicitly when needed

WorkQueue and Interest mirrors/sources use durable, replicated
`AckFlowControl` consumers. Pre-create one when lifecycle, permissions, replay,
filters, or starting position need explicit control, then reference its name
and delivery subject from the source.

`AckFlowControl` requires flow control and heartbeats, acts like `AckAll`, uses
`MaxDeliver: -1`, and does not allow `AckWait` or `BackOff`.

## High-value operations and networking features

### Detect configuration drift

Generate an on-disk configuration digest with `nats-server -t` and compare it
with `varz.config_digest`. A mismatch means the running server has not loaded
the current file.

### Trace without delivering

Publish with `Nats-Trace-Dest` to collect hop events. Add
`Nats-Trace-Only: true` to propagate trace events without delivering the
message. Existing `traceparent` values are preserved.

### Reduce reconnect storms

Set `connect_backoff: true` on routes or gateways for exponential reconnect
delays from one to 30 seconds. For leafnodes, use
`isolate_leafnode_interest` to prevent unnecessary east-west interest
propagation. Leafnode remotes can be toggled, added, or removed by reload.

### Diagnose storage and consensus overload

A filestore write error freezes only its stream, logs `write error`, and fails
health checks; other streams and core traffic continue. Restart the affected
server to recover. Raft bounds proposal growth and can make an overloaded
leader step down, but a cluster remains degraded while a majority lacks
capacity.
