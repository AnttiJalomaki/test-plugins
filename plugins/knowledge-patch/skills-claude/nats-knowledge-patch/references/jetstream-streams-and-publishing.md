# JetStream Streams and Publishing

## Request validation and asset compatibility

### Strict requests

Since 2.12.0, JetStream strict mode is enabled by default. JSON requests with
unknown fields are rejected instead of only producing a log entry. Correct the
client request schema. This server setting temporarily restores the previous
behavior during migration:

```text
jetstream {
  strict: false
}
```

### Asset API levels and managed metadata

NATS Server 2.11.0 assigns the 2.11.x JetStream feature set API support level
`1`. The server advertises its level through `jsz`, `varz`, and
`$JS.API.INFO`. Server-managed assets carry these dynamic metadata keys:

- `_nats.ver`
- `_nats.level`
- `_nats.req.level`

Reconciliation tools must ignore those keys rather than treating them as
configuration drift. Level-dependent fields at this level include nonzero
`PauseUntil` and message-TTL settings.

Features introduced with atomic publishing, counter streams, and schedules
require API level 2.

## Per-message lifetime and deletion markers

### Message TTL

Since 2.11.0, a stream can opt into per-message expiration:

```go
StreamConfig{AllowMsgTTL: true}
```

Publishers then set `Nats-TTL` to integer seconds, a Go duration such as `1h`,
or `never`. `never` exempts the message from both its own expiration and the
stream's `MaxAge`. Invalid and sub-second values reject and discard the
publish. Once `AllowMsgTTL` is enabled, it cannot be disabled.

Sources and mirrors always accept and store an incoming `Nats-TTL` header.
They expire the message only if their own `AllowMsgTTL` is enabled. In
contrast, directly publishing a message with `Nats-TTL` to a stream that has
not enabled the feature is rejected.

### Subject delete markers

`SubjectDeleteMarkerTTL` makes a stream create a marker when age-based
expiration removes the last message for a subject. The marker carries:

```text
Nats-Marker-Reason: MaxAge
Nats-TTL: 1m0s
```

Delete and purge API operations do not create markers, and mirrors cannot
enable this setting.

Delete markers require roll-ups and purge permission. A normal stream create
or update enables `AllowRollup` and clears `DenyPurge` as needed; a pedantic
request fails instead of adjusting those settings. Unless `MaxMsgsPer` is
`1`, `SubjectDeleteMarkerTTL` is also the minimum effective per-message TTL.
A smaller TTL is accepted but clamped, and the stored `Nats-TTL` header is
rewritten to the effective value.

## Ingest backpressure

Since 2.11.0, Core NATS publishing into JetStream is bounded per stream rather
than unlimited. The defaults are:

- `max_buffered_size: 128MB`
- `max_buffered_msgs: 10000`

Configure them in the JetStream block:

```text
jetstream {
  max_buffered_msgs: 50000
  max_buffered_size: 256mib
}
```

Exceeding either limit can drop messages and return
`429 JSStreamTooManyRequests`. JetStream publishes that wait for PubAcks
should not normally reach the bound.

## Atomic and fast batch publishing

### Atomic batches

Since 2.12.0, `AllowAtomicPublish` opts a stream into all-or-nothing batches:

```go
StreamConfig{AllowAtomicPublish: true}
```

The setting requires API level 2, is incompatible with asynchronous
persistence, and cannot be enabled on mirrors. Every publish in a batch uses
the same batch ID and a contiguous sequence:

```text
Nats-Batch-Id: order-42
Nats-Batch-Sequence: 3
Nats-Batch-Commit: 1
```

The first message must be a request. The last stored message receives the
commit header. Only that last message produces a normal PubAck; its `batch`
and `count` fields identify the committed batch.

In 2.12.0, atomic batches have these limits:

- At most 1,000 messages.
- Abandonment after 10 idle seconds.
- No `Nats-Msg-Id`.
- No `Nats-Expected-Last-Msg-Id`.

Abandonment emits the
`io.nats.jetstream.advisory.v1.stream_batch_abandoned` advisory.

### Fast and end-of-batch publishing

Since 2.14.0, `AllowBatchPublish` enables flow-controlled, high-throughput
publishing to replicated and non-replicated streams:

```go
StreamConfig{AllowBatchPublish: true}
```

Fast batches retain per-message consistency checks but do not use atomic
publishing's intermediate staging. Atomic and fast batches can both commit
with an end-of-batch message that is not itself persisted.

## Counter streams

Since 2.12.0, `AllowMsgCounter` makes each subject in a stream an
arbitrary-precision signed counter:

```go
StreamConfig{AllowMsgCounter: true}
```

Every publish must provide a signed-integer `Nats-Incr`. The server replaces
the message body with the new total as `{"val":"..."}` and returns the same
value in the PubAck.

```bash
nats req counter.hits '' -J -H 'Nats-Incr:+2'
```

Counter mode is creation-only, requires Limits retention and API level 2, and
is incompatible with:

- mirrors;
- `DiscardNew`;
- per-message TTL;
- message schedules;
- publishes without a counter increment.

A sourced aggregate counter tracks each upstream total in
`Nats-Counter-Sources` and applies the delta from that source. This keeps
aggregation eventually consistent even when individual source messages were
missed. To reset one source's contribution, publish a compensating negative
increment. A purge does not replicate, while a roll-up would destroy the
combined aggregate.

## Scheduled messages

Since 2.12.0, `AllowMsgSchedules` lets a stored message generate a delayed,
recurring, or sampled message on another subject in the same stream:

```go
StreamConfig{
    AllowMsgSchedules: true,
    AllowMsgTTL:       true,
}
```

Each schedule needs a unique subject. `Nats-Schedule` accepts:

- `@at <RFC3339>`;
- a six-field UTC cron expression;
- a cron alias such as `@hourly`;
- a Go-duration interval such as `@every 5m`.

```bash
nats pub -J schedules.orders.once \
  -H 'Nats-Schedule: @at 2025-10-01T12:00:00Z' \
  -H 'Nats-Schedule-Target: orders' \
  -H 'Nats-Schedule-TTL: 5m' \
  'body'
```

`Nats-Schedule-Source` republishes the latest message from a sampled subject
instead of using a fixed body. A past `@at` time fires immediately.

The two TTL headers have different owners:

- `Nats-TTL` limits the lifetime of the stored schedule.
- `Nats-Schedule-TTL` becomes `Nats-TTL` on each generated message.

Since 2.14.0, `Nats-Schedule-Rollup` similarly applies a roll-up to the
generated message.

Schedule mode requires API level 2. It can be enabled but not disabled on an
existing stream and is rejected on sources and mirrors.

## Source deduplication

Since 2.14.0, a stream with sources can disable source deduplication. A fan-in
stream can instead deduplicate across multiple sources. Choose based on
whether repeated source messages should remain independent or collapse across
the entire fan-in.

## Subject transforms

Since 2.12.0, transforms can derive a value from the entire subject:

- `partition(n)` deterministically chooses a partition using the whole
  subject.
- `random(n)` produces a random number up to `n`.

The older multi-argument `partition(n, …)` form remains available when only
selected subject tokens should influence the partition.

## Replicated deletion and leader changes

Since 2.11.0, deletes in replicated Interest and WorkQueue streams are ordered
through Raft proposals. This improves ordering but can increase replication
traffic.

After a leader change:

- A new leader synchronizes its Raft log before serving reads or writes.
- Replicated consumers redeliver unacknowledged messages.
- Configured consumer start sequences are honored, except for hidden source
  and mirror consumers.
