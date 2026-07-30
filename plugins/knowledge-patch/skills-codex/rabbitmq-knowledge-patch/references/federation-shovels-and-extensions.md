# Federation, Shovels, and extensions

Use this reference for inter-cluster links, in-cluster forwarding, broker
interceptors, plugin integration, and commercial messaging extensions.

## Exchange federation

- Exchange federation supports MQTTv5 consumers.
- RabbitMQ 4.1.8 restores exchange-federation compatibility in mixed
  4.2.x/4.1.x multi-node clusters.
- Configure the AMQP 0-9-1 connection close timeout separately for exchange
  and queue federation. Values are milliseconds and cannot exceed 5000:

```ini
federation.exchanges.connection_close_timeout = 3000
federation.queues.connection_close_timeout = 3000
```

Federation link restart operations in management require the `policymaker`
user tag.

## Local Shovels

The `local` protocol option consumes and publishes within one cluster. These
AMQP 1.0-based Shovels reuse intra-cluster connections and internal
consumption, publishing, and credit-flow APIs instead of opening separate TCP
connections. They cannot connect different clusters.

Resource alarms block direct in-cluster AMQP 0-9-1 Shovel connections just as
they block network Shovel connections. This direct-connection rule does not
describe the newer `local` transport.

Set a stable source consumer identity with `src-consumer-name`. It becomes:

- the AMQP 0-9-1 source consumer tag
- the local source consumer tag
- the AMQP 1.0 source link identifier

Shovel management `DELETE` operations require the `policymaker` tag.

## Native-protocol interceptors

Incoming and outgoing message interceptors cover AMQP 1.0, AMQP 0-9-1,
MQTTv3, and MQTTv5. Plugins can implement validation, annotation, or side
effects. Optional built-in interceptors add outgoing timestamps or the
publishing MQTT client's ID.

## Event exchange

The event exchange plugin can publish internal events using AMQP 1.0 instead
of AMQP 0-9-1. AMQP 1.0 preserves complex properties such as lists and maps.

## Plugin storage and protected resources

- Plugins can mark queues and streams as protected from application deletion.
- During Mnesia-to-Khepri migration, third-party plugins should use the
  dedicated directory that is preserved. Other non-whitelisted directories
  under the node data directory can be removed at migration completion.

## Tanzu JMS queue

The Raft-backed JMS queue is optimized for Qpid JMS and also works with AMQP
1.0, AMQP 0-9-1, STOMP, and MQTT.

- Index fields using queue argument `x-selector-fields` or policy key
  `selector-fields` before using broker-side selectors.
- `QueueBrowser` supports non-destructive selector-based inspection.
- `MessageProducer.setDeliveryDelay(...)` is supported.

## Spark Structured Streaming connector

The `rabbitmq-stream` source reads streams and super streams. It supports:

- starting at head, tail, offset, or timestamp
- field projection
- per-trigger rate limits
- AMQP payload and property access

Relevant options include `uris`, `super.stream`, `starting.offsets`, and
`rmq.stream.select.fields`.

## Stream Browser

The management extension can inspect streams and super streams from an offset,
timestamp, head, or tail. It exposes AMQP 1.0 sections and segment/chunk
layout, and can selectively download message sections.

## Delayed Queue scheduler

The Raft-backed Delayed Queue extension schedules messages through AMQP 1.0
`x-opt-delivery-time` or `x-opt-delivery-delay`, then routes them through
exchanges when the delay expires.

Unlike quorum-queue delayed retries, this supports delayed fan-out. It also
provides browsing, selective purge, and warm-standby replication. The
community `rabbitmq-delayed-message-exchange` plugin is deprecated and
archived.
