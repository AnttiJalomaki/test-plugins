# Consumers, Mirrors, and Sources

## Pull-consumer priority groups

A grouped pull consumer has exactly one `PriorityGroups` entry, uses explicit
acknowledgements, and selects a priority policy. Every pull request supplies
the group.

Only `PriorityTimeout` is updatable. An existing consumer cannot switch into
or out of grouped mode or change its priority policy.

### Overflow

Since 2.11.0, overflow groups delay delivery until a pull request's supplied
threshold is met:

```go
ConsumerConfig{
    PriorityGroups: []string{"jobs"},
    PriorityPolicy: "overflow",
    AckPolicy:      "explicit",
}
```

```json
{"group":"jobs","min_pending":1000,"min_ack_pending":1000}
```

Delivery begins when either `min_pending` or `min_ack_pending` is satisfied.

### Pinned client

Since 2.11.0, `pinned_client` routes delivery to one selected client while
other pulls remain as standbys:

```go
ConsumerConfig{
    PriorityGroups:  []string{"jobs"},
    PriorityPolicy:  "pinned_client",
    PriorityTimeout: 2 * time.Minute,
    AckPolicy:       "explicit",
}
```

This is client orchestration, not a strict guarantee that already in-flight
work belongs only to the selected client. The selected client stores the
`Nats-Pin-Id` response header and sends it as `id` on later pulls. On a `423`
mismatch, clear the stored ID and retry without it.

An administrator can force reselection by publishing `{"group":"jobs"}` to:

```text
$JS.API.CONSUMER.UNPIN.<STREAM>.<CONSUMER>
```

### Prioritized

Since 2.12.0, `prioritized` allows a grouped pull to receive work sooner than
the overflow policy would:

```go
ConsumerConfig{
    PriorityGroups: []string{"jobs"},
    PriorityPolicy: "prioritized",
    AckPolicy:      "explicit",
}
```

The latency benefit comes with the possibility that work flip-flops between
clients.

## Consumer pausing

Since 2.11.0, `PauseUntil` suspends consumer delivery through a deadline. Set
it on creation or through the pause API. Delivery resumes automatically at
the deadline. Heartbeats continue during the pause so clients do not mistake
it for a failed consumer.

## Consumer delivery-state reset

Since 2.14.0, publish an empty payload or `{"seq": N}` to:

```text
$JS.API.CONSUMER.RESET.<STREAM>.<CONSUMER>
```

For example:

```bash
nats req '$JS.API.CONSUMER.RESET.ORDERS.WORKER' '{"seq":100}'
```

An empty request resets pending, redelivery, delivered, and consumer ack-floor
state while retaining the stream ack floor. A positive sequence makes the next
delivered message have a stream sequence of at least `N`.

Arbitrary sequence reset is limited to the `all`, `by_start_sequence`, and
`by_start_time` delivery policies. It cannot violate the configured start
bound. The response contains the updated consumer configuration and state plus
`ResetSeq`.

Clients must tolerate another process resetting a non-ordered consumer. Such a
reset can make its delivery sequence non-monotonic.

## Mirror promotion

Since 2.12.0, a mirror can be promoted into a primary stream for disaster
recovery. Avoid dual ingestion by using this order:

1. Delete the old primary or remove its subjects.
2. Promote the mirror.
3. Configure the promoted stream to listen on the subjects.

Do not attach the subjects before the old primary has stopped ingesting them.

## Reliable WorkQueue and Interest sourcing

Since 2.14.0, mirrors and sources of WorkQueue or Interest streams use durable,
replicated consumers instead of ephemeral `AckNone` consumers. Server-managed
consumers are visible with names such as `JS_MIRROR_<suffix>` and
`JS_SRC_<suffix>`.

They use `AckFlowControl` and acknowledge a message only after the receiving
server has persisted it. An `AckFlowControl` consumer:

- requires flow control and heartbeats;
- behaves like `AckAll`;
- forbids `AckWait` and `BackOff`;
- requires `MaxDeliver: -1`;
- uses `MaxAckPending` to bound outstanding, unacknowledged source messages.

When `MaxAckPending` is reached, sourcing pauses and forces a flow-control
acknowledgement.

### Pre-created source consumers

Pre-create the durable consumer when lifecycle or security control should be
explicit, then reference its name and delivery subject from the source:

```json
{
  "name": "source",
  "consumer": {
    "name": "source-consumer",
    "deliver_subject": "source.consumer.deliver.subject"
  }
}
```

With this arrangement, starting sequence or time and filter settings belong on
the consumer, not on `StreamSource`. The consumer can also specify policies
such as `last_per_subject` and `ReplayPolicy=original`. A WorkQueue stream
still cannot have overlapping filtered consumers.

### Mixed-version and downgrade behavior

During a mixed-version upgrade, an older server can temporarily log an unknown
`sourcing` field while the upgraded server retries the old consumer form.

Downgrading to 2.12 returns these sources to the less reliable ephemeral mode
and can interrupt sourcing during the transition. Existing `AckFlowControl`
consumers remain offline until 2.14.0 is restored.

## ACK and flow-control subjects

Since 2.14.0, servers parse both v1 and domain/account-aware v2 ACK and
flow-control subjects. The server still emits v1 by default. Enable v2
emission with this restart-only feature flag:

```text
feature_flags {
  js_ack_fc_v2: true
}
```

The ACK forms are:

```text
v1: $JS.ACK.<stream>.<consumer>.<delivered>.<stream-seq>.<consumer-seq>.<timestamp>.<pending>
v2: $JS.ACK.<domain>.<account-hash>.<stream>.<consumer>.<delivered>.<stream-seq>.<consumer-seq>.<timestamp>.<pending>
```

Before enabling v2, update custom permissions, imports, and exports that are
narrowly scoped as `$JS.ACK.<stream>.>` or `$JS.FC.<stream>.>`. Catch-all
`$JS.ACK.>` and `$JS.FC.>` rules match both versions.

Client parsers must:

- accept the nine-token v1 form;
- accept v2 forms with 11 or more tokens;
- treat domain `_` as no domain;
- publish the supplied ACK or flow-control reply subject unchanged.

Unknown names inside `feature_flags` are ignored but logged. Feature flags
cannot be reloaded. Remove the entire block before downgrading to a server
version that does not recognize it.
