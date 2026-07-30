# Consumers and sourcing

## Grouped pull consumers

Grouped pull consumers use one `PriorityGroups` entry and explicit
acknowledgements. Every pull identifies the group. The policy controls how
pulls within the group receive work.

### Overflow policy

Since 2.11.0, `PriorityPolicy: "overflow"` waits for thresholds supplied on
each pull:

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

Delivery begins when either supplied `min_pending` or `min_ack_pending`
threshold is met.

### Pinned-client policy

Since 2.11.0, `PriorityPolicy: "pinned_client"` selects one client while other
pulls wait as standbys:

```go
ConsumerConfig{
    PriorityGroups:  []string{"jobs"},
    PriorityPolicy:  "pinned_client",
    PriorityTimeout: 2 * time.Minute,
    AckPolicy:       "explicit",
}
```

This orchestrates selection; it is not a strict exclusivity guarantee for work
already in flight. The selected client stores the `Nats-Pin-Id` response header
and supplies it as `id` on later pulls. After a `423` mismatch, clear the saved
ID and retry without one.

An administrator can force reselection:

```json
{"group":"jobs"}
```

Publish that request to:

```text
$JS.API.CONSUMER.UNPIN.<STREAM>.<CONSUMER>
```

Only `PriorityTimeout` is updatable. A consumer cannot switch into or out of
grouped mode or change its priority policy.

### Prioritized policy

Since 2.12.0, `PriorityPolicy: "prioritized"` lets a grouped pull receive work
sooner than overflow would, at the cost of work potentially flip-flopping
between clients:

```go
ConsumerConfig{
    PriorityGroups: []string{"jobs"},
    PriorityPolicy: "prioritized",
    AckPolicy:      "explicit",
}
```

## Pausing delivery

Since 2.11.0, `PauseUntil` suspends consumer delivery until a deadline. Set it
at creation or through the pause API. Delivery resumes automatically, while
heartbeats continue so clients do not interpret the pause as a failure.

`PauseUntil` is one of the fields gated by the corresponding JetStream asset
API level when it is nonzero.

## Replicated consumer behavior

Since 2.11.0, replicated consumers redeliver unacknowledged messages after
leader changes. Configured consumer start sequences are honored, except for
hidden consumers created for sources or mirrors.

## Reliable WorkQueue and Interest sourcing

Since 2.14.0, mirrors and sources of WorkQueue or Interest streams use durable,
replicated consumers instead of ephemeral `AckNone` consumers. Server-managed
consumers are visible as `JS_MIRROR_<suffix>` or `JS_SRC_<suffix>` and use
`AckFlowControl`. They acknowledge only after the receiving server has
persisted messages.

An `AckFlowControl` consumer:

- requires flow control and heartbeats;
- behaves like `AckAll`;
- forbids `AckWait` and `BackOff`;
- requires `MaxDeliver: -1`.

`MaxAckPending` bounds the number of unacknowledged messages sent before
sourcing pauses and forces a flow-control acknowledgement.

### Pre-created source consumers

For explicit lifecycle or security control, pre-create the durable consumer and
reference its name and delivery subject:

```json
{
  "name": "source",
  "consumer": {
    "name": "source-consumer",
    "deliver_subject": "source.consumer.deliver.subject"
  }
}
```

With a pre-created consumer, start sequence or time and filter settings belong
on the consumer rather than `StreamSource`. This also permits policies such as
`last_per_subject` and `ReplayPolicy=original`. A WorkQueue stream still cannot
have overlapping filtered consumers.

During a mixed-version upgrade, older peers may temporarily log an unknown
`sourcing` field while an upgraded server retries the old consumer form.
Downgrading to 2.12 returns these sources to the less reliable ephemeral mode,
may interrupt sourcing during the transition, and leaves `AckFlowControl`
consumers offline until 2.14 is restored.

## Resetting consumer delivery state

Since 2.14.0, publish an empty payload or `{"seq": N}` to:

```text
$JS.API.CONSUMER.RESET.<STREAM>.<CONSUMER>
```

```bash
nats req '$JS.API.CONSUMER.RESET.ORDERS.WORKER' '{"seq":100}'
```

An empty payload resets pending, redelivery, delivered, and consumer ack-floor
state while retaining the stream ack floor. A positive sequence makes the next
delivered message have a stream sequence of at least `N`.

Arbitrary sequence reset is limited to `all`, `by_start_sequence`, and
`by_start_time` delivery policies and cannot violate the configured start
bound. The response includes the updated consumer configuration, state, and
`ResetSeq`. Clients must tolerate another process resetting a non-ordered
consumer and making its delivery sequence non-monotonic.

## Domain-aware ACK and flow-control subjects

Since 2.14.0, servers parse both v1 and domain/account-aware v2 ACK and
flow-control subjects. The server still emits v1 by default. Enable v2 emission
with the restart-only feature flag:

```text
feature_flags {
  js_ack_fc_v2: true
}
```

```text
v1: $JS.ACK.<stream>.<consumer>.<delivered>.<stream-seq>.<consumer-seq>.<timestamp>.<pending>
v2: $JS.ACK.<domain>.<account-hash>.<stream>.<consumer>.<delivered>.<stream-seq>.<consumer-seq>.<timestamp>.<pending>
```

Before v2 becomes the default, update custom permissions or imports and exports
scoped as `$JS.ACK.<stream>.>` or `$JS.FC.<stream>.>`. Catch-all `$JS.ACK.>`
and `$JS.FC.>` rules already cover both forms.

Client parsers must accept the 9-token v1 form and v2 forms with 11 or more
tokens. Treat domain `_` as no domain, and publish the supplied ACK or
flow-control reply subject unchanged.

Unknown feature-flag names are ignored but logged. Flags cannot be reloaded.
Remove the entire `feature_flags` block before downgrading to a server that
does not recognize it.
