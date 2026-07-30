# Protocols, Clustering, and Federation

Use this reference for protocol wire behavior, peer discovery, stream
replication networking, federation, shovels, and WebSocket transport.

## Cluster Formation and Peer Discovery

Khepri's default cluster-formation timeout matches Mnesia at five minutes as
of batch `4.0.6`.

Peer discovery accepts a positive retry count or infinite retries:

```ini
cluster_formation.discovery_retry_limit = infinity
```

Consul can perform discovery without registering services when another system
such as Nomad owns registration:

```ini
cluster_formation.registration.enabled = false
```

A reset former Mnesia cluster member now attempts to leave the cluster and
retries joining, matching Khepri behavior.

The Kubernetes discovery plugin no longer relies on the Kubernetes API. On
first formation it tries to join node index `0` as the seed; this remains
backwards compatible.

The AWS peer-discovery plugin uses IPv6 discovery endpoints in IPv6-only
environments starting in 4.1.7.

## Stream Replication Networking

Stream replicas can use IPv6 through `advanced.config`:

```erlang
[
  {osiris, [{replica_ip_address_family, inet6}]}
].
```

Starting in 4.1.7, the stream replication IPv4/IPv6 family can also be
selected in `rabbitmq.conf`.

## AMQP 1.0

### Sender, routing, and node behavior

When a sender omits the AMQP 1.0 header section in batch `4.2.0`, RabbitMQ
applies the specification default `durable=false`. Persistent applications
must send a header with `durable=true`.

An AMQP 1.0 publisher can attach a list of string routing keys in the `x-cc`
message annotation, equivalent to the AMQP 0-9-1 `CC` header.

RabbitMQ honors `dynamic` on sources and targets, allowing clients to create
exclusive queues dynamically for RPC and similar workloads.

The Erlang AMQP 1.0 client declares durable entities only. Purging a
nonexistent queue through that client returns `404`.

The event exchange plugin can publish internal events as AMQP 1.0 instead of
AMQP 0-9-1, preserving list and map properties.

### Authentication and outcomes

AMQP 1.0 JWT renewal behavior is covered in
[configuration-auth-and-security.md](configuration-auth-and-security.md).

A `Rejected` outcome in 4.3 identifies the rejecting queue and reason, such as
a length limit or unavailable queue. A publisher routed to several queues can
therefore locate the failed target.

### Direct Reply-To

AMQP 1.0 supports Direct Reply-To in 4.2, including RPC where requester and
responder use AMQP 1.0 and AMQP 0-9-1 in either combination.

For an AMQP 0-9-1 responder using `mandatory`, an
`amq.rabbitmq.reply-to.*` destination is considered routed without checking
whether the requester is still consuming. RabbitMQ does not emit
`basic.return` when it is the only target.

## AMQP 0-9-1 Connection Limits

The pre-authentication maximum frame size is 8192 bytes in the 4.1 guidance,
so a client `frame_max` override must be at least 8192. The recommended choice
is the server default, 131072. Node.js `amqplib` must be 0.10.7 or later or
configured with a larger value.

AMQP listener user-space TCP buffers are auto-tuned, and
`tcp_listen_options.buffer` is ignored. Kernel `recbuf` and `sndbuf` are
unaffected.

## MQTT and WebSocket Behavior

The default MQTT Maximum Packet Size is 16 MiB instead of 256 MiB. Override it
with `mqtt.max_packet_size_authenticated`, never above `max_message_size`,
which also defaults to 16 MiB.

Exchange federation supports MQTTv5 consumers.

Starting in 4.1.7, Web MQTT accepts the `mqttv3.1` WebSocket subprotocol as
well as `mqtt`.

Starting in 4.1.8, authorization failures disconnect MQTT clients by default.
To preserve the connection and return the protocol-level error:

```ini
mqtt.disconnect_on_unauthorized = false
```

In 4.2, Web MQTT and Web STOMP enable HTTP/2 for WebSocket connections by
default.

In 4.3, an MQTT 5.0 publish rejected because the target queue reached its
maximum length receives `Quota exceeded` in `PUBACK`.

Web MQTT bounds decompressed frames at
`mqtt.max_packet_size_unauthenticated`, raises the bound after a valid
`CONNECT`, and enforces `login_timeout`. Configure origin validation with
`web_mqtt.allow_origins` and `web_stomp.allow_origins`.

STOMP subscriptions for destinations that formerly created non-exclusive
transient queues now create exclusive queues.

## Federation

RabbitMQ 4.1.8 restores exchange-federation compatibility in mixed
4.2.x/4.1.x multi-node clusters.

Configure separate AMQP 0-9-1 federation connection-close timeouts:

```ini
federation.exchanges.connection_close_timeout = 3000
federation.queues.connection_close_timeout = 3000
```

Values are milliseconds and cannot exceed 5000.

## Shovels and Interceptors

RabbitMQ 4.2 message interceptors cover AMQP 1.0, AMQP 0-9-1, MQTTv3, and
MQTTv5 in both incoming and outgoing directions. Plugins can validate,
annotate, or trigger side effects. Optional built-ins add outgoing timestamps
or the publishing MQTT client's ID.

Shovels support the `local` protocol for consuming and publishing within one
cluster. These AMQP 1.0-based shovels reuse intra-cluster connections and
internal consumption, publication, and credit-flow APIs instead of TCP
connections. They cannot connect different clusters.

Resource alarms block direct in-cluster AMQP 0-9-1 shovel connections just as
they block network shovel connections. This applies to direct connections,
not to the `local` protocol.

In 4.3, `src-consumer-name` sets the AMQP 0-9-1 or local source consumer tag,
or the AMQP 1.0 source link identifier.
