# Streams and publishing

## Strict request validation

Since 2.12.0, JetStream strict mode is enabled by default. JSON requests with
unknown fields are rejected rather than only logged. Correct clients that send
invalid fields. During migration, the old behavior can be restored temporarily:

```text
jetstream {
  strict: false
}
```

## Per-message TTL

Since 2.11.0, streams opt in with `AllowMsgTTL`. Publishers may then set
`Nats-TTL` to integer seconds, a Go duration such as `1h`, or `never`.

```go
StreamConfig{AllowMsgTTL: true}
```

- `never` exempts the message from the stream's `MaxAge`.
- Invalid or sub-second values reject and discard the publish.
- `AllowMsgTTL` cannot be disabled after it is enabled.
- A direct publish with `Nats-TTL` to a stream without the feature is rejected.
- Sources and mirrors always accept and store an incoming `Nats-TTL`, but
  expire that message only if their own `AllowMsgTTL` is enabled.

### Subject delete markers

`SubjectDeleteMarkerTTL` creates a marker when age-based removal deletes the
last message for a subject. The marker has its own TTL and reason:

```text
Nats-Marker-Reason: MaxAge
Nats-TTL: 1m0s
```

Delete and purge API calls do not create markers, and mirrors cannot enable the
setting. Markers require roll-ups and permission to purge. Normal create or
update requests enable `AllowRollup` and clear `DenyPurge`; pedantic requests
fail instead.

Unless `MaxMsgsPer` is `1`, `SubjectDeleteMarkerTTL` is also the minimum
effective per-message TTL. Smaller values are accepted but clamped, and the
stored `Nats-TTL` header is rewritten.

## Stream ingest backpressure

Since 2.11.0, Core NATS publishes entering JetStream are bounded per stream:

```text
jetstream {
  max_buffered_msgs: 50000
  max_buffered_size: 256mib
}
```

The defaults are `10,000` messages and `128MB`; NATS 2.10 was unlimited.
Crossing either limit can drop messages and return
`429 JSStreamTooManyRequests`. JetStream publishes that wait for PubAcks should
not normally reach these limits.

## Atomic stream batches

Since 2.12.0, `AllowAtomicPublish` enables all-or-nothing batches. It requires
JetStream API level 2, is incompatible with asynchronous persistence, and
cannot be enabled on mirrors.

```go
StreamConfig{AllowAtomicPublish: true}
```

Every publish uses the same batch ID and a contiguous sequence. The first
publish must be a request. The final stored message adds the commit header:

```text
Nats-Batch-Id: order-42
Nats-Batch-Sequence: 3
Nats-Batch-Commit: 1
```

Only the final message receives a normal PubAck. Its `batch` and `count` fields
identify the committed batch. In 2.12.0, the limit is 1,000 messages and the
batch expires after 10 idle seconds. Atomic batches reject `Nats-Msg-Id` and
`Nats-Expected-Last-Msg-Id`. Abandonment emits
`io.nats.jetstream.advisory.v1.stream_batch_abandoned`.

## Fast and end-of-batch publishing

Since 2.14.0, `AllowBatchPublish` provides flow-controlled, high-throughput
publishing to replicated and non-replicated streams. It keeps per-message
consistency checks but does not use atomic publishing's intermediate staging.

```go
StreamConfig{AllowBatchPublish: true}
```

Atomic and fast batches can commit with an end-of-batch message that is not
itself persisted.

## Distributed counter streams

Since 2.12.0, `AllowMsgCounter` turns every subject in a stream into an
arbitrary-precision signed counter. Every publish must carry `Nats-Incr` in
signed-integer form. The server replaces the body with the new
`{"val":"..."}` total and returns that value in the PubAck.

```go
StreamConfig{AllowMsgCounter: true}
```

```bash
nats req counter.hits '' -J -H 'Nats-Incr:+2'
```

Counter mode:

- is set only at stream creation;
- requires Limits retention and API level 2;
- is incompatible with mirrors, DiscardNew, per-message TTL, message
  schedules, and publishes without a counter increment.

For sourced aggregate counters, `Nats-Counter-Sources` tracks each upstream
total and the receiver applies its delta. This makes aggregation eventually
consistent even after source messages are missed. Reset one sourced
contribution with a compensating negative increment: purge does not replicate,
and roll-up would destroy the combined counter.

## Message schedules

Since 2.12.0, `AllowMsgSchedules` lets one stored message produce a delayed,
recurring, or sampled message on another subject in the same stream. Each
schedule requires a unique subject.

```go
StreamConfig{
    AllowMsgSchedules: true,
    AllowMsgTTL:       true,
}
```

`Nats-Schedule` accepts:

- `@at <RFC3339>`;
- a six-field UTC cron expression or alias such as `@hourly`;
- a Go-duration interval such as `@every 5m`.

```bash
nats pub -J schedules.orders.once \
  -H 'Nats-Schedule: @at 2025-10-01T12:00:00Z' \
  -H 'Nats-Schedule-Target: orders' \
  -H 'Nats-Schedule-TTL: 5m' \
  'body'
```

`Nats-Schedule-Source` republishes the latest message on a sampled subject.
`Nats-Schedule-TTL` becomes `Nats-TTL` on generated messages, while `Nats-TTL`
on the schedule record limits the schedule itself. Past `@at` values fire
immediately.

Schedule mode requires API level 2, may be enabled but not disabled on an
existing stream, and is rejected on sources and mirrors. Since 2.14.0,
`Nats-Schedule-Rollup` applies a roll-up to the generated message, analogous to
`Nats-Schedule-TTL`.

## Mirrors, promotion, and source deduplication

Since 2.12.0, a mirror can be promoted to a primary stream for disaster
recovery. Delete the old primary or remove its subjects first, promote the
mirror second, and only then configure the promoted stream to listen on those
subjects. This order avoids two primaries ingesting the same traffic.

Since 2.14.0, streams with sources can disable deduplication, and fan-in streams
can deduplicate across multiple sources.

## Whole-subject transforms

Since 2.12.0, subject transforms include:

- `partition(n)`, which deterministically derives a partition from the whole
  subject;
- `random(n)`, which produces a random number up to `n`.

The older multi-argument `partition(n, …)` remains available when selected
subject tokens should drive partitioning.
