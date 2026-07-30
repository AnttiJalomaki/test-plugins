# Queues, Streams, and Messaging

Use this reference for queue declarations, quorum delivery semantics, stream
filtering, Single Active Consumer control, delayed delivery, and commercial
queue or stream capabilities.

## Queue and Stream Protection

Virtual hosts can be protected through metadata, and plugins can mark queues
or streams as protected from application deletion. User-level HTTP API
protection is covered separately in
[configuration-auth-and-security.md](configuration-auth-and-security.md).

RabbitMQ 4.2 lets administrators disable individual queue types. A client
cannot declare a new queue or stream whose type is disabled.

## Stream Retention and Filtering

The command `rabbitmq-streams set_stream_retention_policy` is a no-op as of
batch `4.0.6`. Set stream retention through a policy.

The filters introduced in batch `4.1-guides` support the `properties` and
`application-properties` filter types from AMQP Filter Expressions Working
Draft 09. Concurrent consumers can select different stream subsets without
losing order.

An AMQP 1.0 properties or application-properties filter can inspect no more
than 16 properties.

RabbitMQ 4.2 adds broker-side SQL expressions for exact per-message stream
filtering. Use a Bloom-filter value to skip entire chunks before SQL inspects
standard message fields and application properties:

```java
String sql =
    "properties.subject = 'order.created' AND " +
    "region IN ('AMER', 'EMEA', 'APJ')";

Consumer consumer = connection.consumerBuilder()
    .queue(STREAM_NAME)
    .stream()
    .offset(FIRST)
    .filterValues("order.created")
    .filter()
        .sql(sql)
    .stream()
    .builder()
    .messageHandler((ctx, msg) -> ctx.accept())
    .build();
```

This capability is attributed to batch `4.2-guides`.

## Quorum Queue Maintenance

Starting in 4.1.1, force matching quorum queues to checkpoint and remove
on-disk segment files where possible:

```shell
rabbitmq-queues force_checkpoint \
  --vhost-pattern "vhost-pattern" \
  --queue-pattern "queue-pattern"
```

In 4.3, a delivery limit for a quorum queue can be changed by policy without
redeclaring the queue.

Purging a quorum queue now also removes at-least-once dead-lettered messages
still pending delivery.

## Single Active Consumer

Starting in 4.1.2, an operator can force one consumer to become active in a
stream Single Active Consumer group:

```shell
rabbitmq-streams activate_stream_consumer \
  --stream "stream-name" \
  --reference "consumer-reference"
```

In 4.3, an AMQP 1.0 quorum-queue consumer receives Single Active Consumer
active/inactive changes immediately in flow-frame properties instead of
inferring whether its link is selected.

## Priorities and Delayed Retries

Batch `4.3-guides` changes quorum queues from the earlier two-level priority
scheme to 32 strict priority levels. Every higher-priority message is
delivered before every lower-priority message; the management UI reports
counts per priority. Do not expect the earlier 2:1 interleaving of high and
normal messages.

Quorum queues can defer returned messages without a dead-letter cycle or an
external scheduler. Configure:

- `x-delayed-retry-type`: `disabled` (default), `all`, `returned`, or `failed`
- `x-delayed-retry-min`: minimum delay in milliseconds
- `x-delayed-retry-max`: maximum delay in milliseconds; defaults to the
  minimum

Equivalent policy keys omit the `x-` prefix. Delay is:

```text
min(delayed-retry-min * delivery-count, delayed-retry-max)
```

An AMQP 1.0 modified outcome can override the per-message delay with the
Unix-millisecond `x-opt-delivery-time` annotation.

## Delivery Count and Poison-Message Semantics

Quorum queues increment `acquired-count` for every requeue, but increment
`delivery-count` only for failed attempts. Poison-message handling uses
`delivery-count`, so non-failed returns can be unlimited.

The following are non-failures:

- AMQP 1.0 `released`
- AMQP 1.0 `modified` with `delivery-failed=false`
- AMQP 0-9-1 `basic.nack`
- An intra-cluster partition that makes the consumer node suspect
- Consumer timeout

The following count as failures:

- AMQP 1.0 `rejected`
- AMQP 1.0 `modified` with `delivery-failed=true`
- AMQP 0-9-1 `basic.reject`
- Client or connection loss

## Consumer Timeout Semantics

Consumer timeouts are evaluated by quorum queues and Tanzu JMS queues, not by
classic queues or streams.

For a timed-out AMQP 1.0 delivery, the broker releases the delivery without
detaching the link. For AMQP 0-9-1, the broker cancels only the affected
consumer when `consumer_cancel_notify` is supported; otherwise it closes the
channel.

Configuration precedence is:

1. Consumer `x-consumer-timeout`
2. Queue `x-consumer-timeout`
3. Policy `consumer-timeout`
4. Global `consumer_timeout`

The global default is 1,800,000 ms.

Starting in 4.3.0, a quorum queue waits 60 seconds by default before returning
messages after a consumer's node becomes unreachable during a partition.
Override this globally with `consumer_disconnected_timeout`, by policy with
`consumer-disconnected-timeout`, or per queue with
`x-consumer-disconnected-timeout`.

## Exchange Routing

In 4.3, `x-modulus-hash` moves from the sharding plugin into the core. With a
stable binding set, its routing distribution remains stable across node
restarts.

## Commercial Tanzu Capabilities

### JMS queue type

The Tanzu edition adds a Raft-backed JMS queue optimized for Qpid JMS and
usable through AMQP 1.0, AMQP 0-9-1, STOMP, and MQTT. It supports:

- Broker-side selectors after fields are indexed with queue argument
  `x-selector-fields` or policy key `selector-fields`
- Non-destructive `QueueBrowser` inspection with selectors
- `MessageProducer.setDeliveryDelay(...)`

### Spark stream connector

The Tanzu RabbitMQ Stream connector for Spark Structured Streaming reads
streams and super streams. It supports starting at head, tail, an offset, or
a timestamp; field projection; per-trigger rate limits; and AMQP payload and
property access. Options include `uris`, `super.stream`, `starting.offsets`,
and `rmq.stream.select.fields`.

### Stream Browser

The Stream Browser management plugin inspects streams and super streams from
an offset, timestamp, head, or tail. It exposes AMQP 1.0 sections and
segment/chunk layout and can selectively download message sections.

### Delayed Queue plugin

The community `rabbitmq-delayed-message-exchange` plugin is deprecated and
archived. Tanzu 4.3 provides a Raft-backed Delayed Queue plugin that schedules
using AMQP 1.0 `x-opt-delivery-time` or `x-opt-delivery-delay`, then routes
through exchanges when the delay expires.

Unlike quorum-queue delayed retries, this supports delayed fan-out. It also
supports browsing, selective purge, and warm-standby replication.
