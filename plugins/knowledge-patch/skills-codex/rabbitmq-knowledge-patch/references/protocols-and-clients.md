# Protocols and clients

Use this reference for wire-level behavior, client compatibility, and
cross-protocol interactions.

## Contents

- [AMQP 1.0 sending and outcomes](#amqp-10-sending-and-outcomes)
- [AMQP 1.0 clients and declarations](#amqp-10-clients-and-declarations)
- [Direct Reply-To](#direct-reply-to)
- [AMQP 0-9-1 compatibility](#amqp-0-9-1-compatibility)
- [MQTT](#mqtt)
- [Web MQTT and Web STOMP](#web-mqtt-and-web-stomp)
- [Resource-alarm behavior](#resource-alarm-behavior)
- [TCP buffers](#tcp-buffers)
- [Single Active Consumer state](#single-active-consumer-state)

## AMQP 1.0 sending and outcomes

### Durability defaults

If a sender omits the AMQP 1.0 header section, RabbitMQ follows the protocol
default and treats `durable` as `false`. Send a header with `durable=true` when
persistence is required.

### Multiple routing keys

Put a list of string routing keys in the `x-cc` message annotation. This is the
AMQP 1.0 equivalent of the AMQP 0-9-1 `CC` header.

### Dynamic nodes

RabbitMQ honors the `dynamic` field on AMQP 1.0 sources and targets, allowing
clients to create dynamic exclusive queues for uses such as RPC.

### Rejection details

A `Rejected` outcome identifies the rejecting queue and the reason, such as a
length limit or unavailable queue. A publisher routed to several queues can
therefore identify which target failed.

### Event exchange payloads

The event exchange plugin can publish internal events as AMQP 1.0 instead of
AMQP 0-9-1, preserving complex properties such as lists and maps.

## AMQP 1.0 clients and declarations

- AMQP 1.0 clients can replace an OAuth JWT before expiry without
  disconnecting. RabbitMQ closes the connection if no replacement arrives in
  time.
- The Erlang AMQP 1.0 client can declare durable entities only.
- Purging a nonexistent queue with the Erlang AMQP 1.0 client returns `404`.
- AMQP 1.0 property and application-property stream filters accept no more
  than 16 properties.

## Direct Reply-To

AMQP 1.0 supports Direct Reply-To, including RPC where requester and responder
use AMQP 1.0 and AMQP 0-9-1 in either combination.

When an AMQP 0-9-1 responder publishes with `mandatory`, an
`amq.rabbitmq.reply-to.*` target counts as routed without checking whether the
requester is still consuming. RabbitMQ does not send `basic.return` when that
is the only target.

## AMQP 0-9-1 compatibility

### Pre-authentication frames

The pre-authentication maximum frame size is 8192 bytes. A client's
`frame_max` override must be at least that large; leaving the server default
of 131072 is recommended. Node.js `amqplib` clients need version 0.10.7 or
later, or an explicitly larger value.

### Permissions after credential refresh

Refreshing connection credentials clears the permission cache and immediately
revalidates consumer permissions. Passive queue and exchange declarations
require `configure` permission, like regular declarations.

## MQTT

### Packet and message size

The default MQTT Maximum Packet Size is 16 MiB rather than 256 MiB. Override
it with `mqtt.max_packet_size_authenticated`, but never set it above
`max_message_size`, whose default is also 16 MiB.

### Authorization failure

`mqtt.disconnect_on_unauthorized` defaults to `true`. Set it to `false` to keep
the connection open and send the protocol-level error:

```ini
mqtt.disconnect_on_unauthorized = false
```

### MQTT 5 queue feedback

When a target queue refuses a publish because its maximum length is reached,
an MQTT 5 publisher receives the `Quota exceeded` reason code in `PUBACK`.

### Federation

MQTTv5 consumers can use exchange federation.

## Web MQTT and Web STOMP

- The Web MQTT plugin accepts the `mqttv3.1` WebSocket subprotocol as well as
  `mqtt`.
- Web MQTT bounds decompressed frames. The limit starts at
  `mqtt.max_packet_size_unauthenticated` and rises after a successful
  `CONNECT`.
- Web MQTT enforces `login_timeout`.
- Configure origin validation with `web_mqtt.allow_origins` and
  `web_stomp.allow_origins`.
- Web MQTT and Web STOMP enable HTTP/2 for WebSocket connections by default.
- STOMP subscriptions whose destination formerly used a non-exclusive
  transient queue now use an exclusive queue.

## Resource-alarm behavior

MQTT, STOMP, and Web MQTT connections stay blocked until every active memory or
disk alarm clears. Clearing one alarm does not unblock a connection while
another remains active.

## TCP buffers

AMQP listener user-space TCP buffers are auto-tuned, so
`tcp_listen_options.buffer` is ignored. Kernel-level `recbuf` and `sndbuf`
remain configurable.

## Single Active Consumer state

AMQP 1.0 quorum-queue consumers receive immediate active/inactive state
changes through flow-frame properties instead of inferring whether a link was
selected.
