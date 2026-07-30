# Microservices and Observability

Use this reference for transport state, driver access, structured logging, and
transport-specific capabilities. It incorporates the `11.0.0` and
`11.1.x-selected` guidance.

## Transport observability and native access

Microservice client and server abstractions expose three capabilities:

- `status` observes transport state.
- `on` subscribes to native-driver events.
- `unwrap` accesses the underlying driver.

Use `status` and `on` for lifecycle monitoring without abandoning the Nest
abstraction. Use `unwrap` only when a driver-specific operation cannot be
expressed through that abstraction.

## Structured console logging

`ConsoleLogger` can emit JSON:

```typescript
const app = await NestFactory.create(AppModule, {
  logger: new ConsoleLogger({ json: true }),
});
```

This supplies structured console output without replacing the built-in logger
with a separate implementation.

## Dependency-injected microservice options

Microservice configuration can be resolved through the dependency-injection
container. Transport options may depend on registered providers instead of
being constructed entirely outside Nest.

Use this capability when configuration, credentials, or another transport
dependency is already represented by a provider. Keep the resulting options
compatible with the selected transport.

## NATS

NATS message handlers can choose queues individually. Queue selection no longer
has to be one value for the entire server, so handlers can participate in
different queue groups.

The NATS transporter also has an optional graceful-shutdown path. Use it when
the service must avoid immediate transporter teardown during application
shutdown.

## TCP

The TCP transporter accepts an operating-system-selected port, enabling
ephemeral-port startup. This is useful when the process should bind an
available port rather than a fixed one.

It also accepts a configurable maximum packet-buffer size. Set an explicit
bound when the service must limit how much packet data can be buffered.

## RabbitMQ

RabbitMQ microservices support topic exchanges (`11.1.x-selected`). Use the RMQ
transport's topic-based routing when routing keys need topic matching rather
than a non-topic exchange pattern.

