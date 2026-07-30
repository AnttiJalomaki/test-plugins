# Queues, streams, and exchanges

Use this reference for declaration constraints, quorum-queue delivery
semantics, stream consumption, and exchange controls.

## Contents

- [Resource protection and declaration controls](#resource-protection-and-declaration-controls)
- [Quorum diagnosis and maintenance](#quorum-diagnosis-and-maintenance)
- [Priority behavior](#priority-behavior)
- [Native delayed retries](#native-delayed-retries)
- [Delivery and poison-message counts](#delivery-and-poison-message-counts)
- [Consumer timeouts](#consumer-timeouts)
- [Single Active Consumer](#single-active-consumer)
- [Stream filters](#stream-filters)
- [Stream retention and replication](#stream-retention-and-replication)
- [Commercial queue and stream facilities](#commercial-queue-and-stream-facilities)

## Resource protection and declaration controls

- Virtual hosts can carry metadata that protects them from deletion.
- Plugins can mark queues and streams as protected so applications cannot
  delete them.
- Administrators can disable individual queue types. Clients then cannot
  declare new queues or streams of those types.
- Starting in 4.1.1, a new virtual host has its default queue type injected
  into metadata, keeping definition export methods consistent.
- RabbitMQ 4.1.4 adds a cluster-wide `cluster_exchange_limit`. It counts
  application-declared and protocol-standard predeclared exchanges and must
  have the same value on every node:

```ini
cluster_exchange_limit = 200
```

- Environments whose load balancers cannot preserve locality can reject new
  local-random exchange declarations:

```ini
exchange_types.local_random.enabled = false
```

- `x-modulus-hash` is now a core exchange type rather than part of the
  sharding plugin. With a stable binding set, its distribution remains stable
  across node restarts.

## Quorum diagnosis and maintenance

Check matching quorum queues for an elected leader:

```shell
rabbitmq-diagnostics check_for_quorum_queues_without_an_elected_leader \
  --vhost "vh-1" "^naming-pattern"
```

Use `--across-all-vhosts ".*"` for a cluster-wide check, but expect it to be
expensive on clusters with many quorum queues.

Starting in 4.1.1, force matching queues to checkpoint and remove on-disk
segment files where possible:

```shell
rabbitmq-queues force_checkpoint \
  --vhost-pattern "vhost-pattern" \
  --queue-pattern "queue-pattern"
```

A quorum queue's delivery limit can be changed by policy without redeclaring
the queue. Purging a quorum queue also removes at-least-once dead-lettered
messages still pending delivery.

## Priority behavior

Quorum queues use 32 strict priority levels: all higher-priority messages are
delivered before lower-priority messages. This replaces the earlier two-level
model, which interleaved high and normal messages at a 2:1 ratio. The
management UI reports counts per priority.

## Native delayed retries

Quorum queues can hold returned messages and redeliver them later without a
dead-letter cycle or an external scheduler. Configure arguments:

- `x-delayed-retry-type`: `disabled` by default, or `all`, `returned`, or
  `failed`
- `x-delayed-retry-min`: minimum delay in milliseconds
- `x-delayed-retry-max`: maximum delay in milliseconds; defaults to the
  minimum

Policy keys use the same names without `x-`. The delay is:

```text
min(delayed-retry-min * delivery-count, delayed-retry-max)
```

AMQP 1.0 can override the delay for a message using the Unix-millisecond
`x-opt-delivery-time` annotation on a modified outcome.

## Delivery and poison-message counts

Every requeue increments `acquired-count`, but only failed delivery attempts
increment `delivery-count`. Poison-message handling uses `delivery-count`;
ordinary returns can therefore be unlimited.

Non-failures include:

- AMQP 1.0 `released`
- AMQP 1.0 `modified` with `delivery-failed=false`
- AMQP 0-9-1 `basic.nack`
- a partition that makes the consumer node suspect
- consumer timeouts

Failures include:

- AMQP 1.0 `rejected`
- AMQP 1.0 `modified` with `delivery-failed=true`
- AMQP 0-9-1 `basic.reject`
- client or connection loss

## Consumer timeouts

### Delivery acknowledgement timeout

The effective value is selected in this order:

1. consumer `x-consumer-timeout`
2. queue `x-consumer-timeout`
3. policy `consumer-timeout`
4. global `consumer_timeout`

The global default is `1800000` milliseconds. Quorum queues and commercial JMS
queues evaluate this timeout; classic queues and streams do not. For a timed
out AMQP 1.0 delivery, the broker releases the delivery without detaching the
link. AMQP 0-9-1 cancels only the consumer when `consumer_cancel_notify` is
supported; otherwise it closes the channel.

### Disconnected-consumer timeout

When a consumer node becomes unreachable in a partition, a quorum queue waits
60 seconds by default before returning messages. Override it with global
`consumer_disconnected_timeout`, policy `consumer-disconnected-timeout`, or
queue argument `x-consumer-disconnected-timeout`.

## Single Active Consumer

- Starting in 4.1.2, an operator can force a chosen stream consumer to become
  active:

```shell
rabbitmq-streams activate_stream_consumer \
  --stream "stream-name" \
  --reference "consumer-reference"
```

- AMQP 1.0 consumers on a quorum queue receive immediate active/inactive
  Single Active Consumer state through flow-frame properties.

## Stream filters

### Property filters

AMQP 1.0 stream consumers can use the `properties` and
`application-properties` filters from AMQP Filter Expressions Working Draft 09.
Concurrent consumers can select different ordered subsets of a stream. A
filter cannot inspect more than 16 properties.

### SQL filters

Broker-side SQL expressions can inspect standard fields and application
properties. Combine a Bloom-filter value with SQL so the Bloom filter can
skip entire chunks before the broker evaluates messages:

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

## Stream retention and replication

- Configure stream retention with a policy; the old
  `set_stream_retention_policy` command has no effect.
- Stream replication can use IPv6 through `advanced.config`:

```erlang
[
  {osiris, [{replica_ip_address_family, inet6}]}
].
```

- Starting in 4.1.7, select the replication IP family in `rabbitmq.conf` as
  well as through `advanced.config`.

## Commercial queue and stream facilities

- The Tanzu JMS queue is Raft-backed and optimized for Qpid JMS while remaining
  usable through AMQP 1.0, AMQP 0-9-1, STOMP, and MQTT.
- Index selector fields with queue argument `x-selector-fields` or policy key
  `selector-fields`; the queue supports broker-side selectors,
  selector-aware non-destructive `QueueBrowser` inspection, and
  `MessageProducer.setDeliveryDelay(...)`.
- Stream Browser can inspect streams and super streams from an offset,
  timestamp, head, or tail; expose AMQP 1.0 sections and segment/chunk layout;
  and selectively download message sections.
