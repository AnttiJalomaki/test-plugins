# Transports and Operations

## Transport state, events, and native access

Starting with 11.0.0, microservice client and server abstractions expose three
capabilities without requiring callers to leave Nest's abstraction:

- `status` exposes transport state.
- `on` subscribes to native-driver events.
- `unwrap` returns the underlying driver when native APIs are required.

Prefer the abstract state and event APIs where they suffice. Use `unwrap` when a
driver-specific feature is necessary.

## Dependency-injected transport options

Microservice configuration can be resolved through the dependency-injection
container as of 11.0.0. Transport options may depend on registered providers
instead of being constructed completely outside Nest.

Use this capability when configuration or credentials already come from
providers and the transporter must be assembled from those injected values.

## NATS

NATS message handlers can choose queues individually as of 11.0.0. Queue
selection no longer has to be one setting for the entire server.

The NATS transporter also has an optional graceful-shutdown path. Applications
can use it instead of forcing immediate teardown for the whole server.

## TCP

The TCP transporter additions in 11.0.0 support:

- an operating-system-selected port, enabling ephemeral-port startup;
- a configurable maximum packet-buffer size, setting an explicit bound on
  buffered packets.

Use an ephemeral port when the actual bound port can be discovered after
startup. Set the buffer maximum when the application needs a defined packet
buffer bound.

## RabbitMQ topic exchanges

RabbitMQ microservices support topic exchanges in 11.1.x-selected. Applications
can use topic-based routing through Nest's RMQ transport.

## WebSocket extension points

The 11.0.0 WebSocket changes provide two extension points:

- WebSocket errors can retain a cause.
- The `ws` adapter exposes a message-parser extension point for custom wire
  formats.

Preserve the underlying cause when wrapping an error. Supply a custom parser
when the adapter must decode a wire format other than its default format.

## Structured console logging

`ConsoleLogger` can emit JSON output as of 11.0.0:

```typescript
const app = await NestFactory.create(AppModule, {
  logger: new ConsoleLogger({ json: true }),
});
```

Pass the configured logger at application creation when downstream log handling
expects structured JSON records.

## Terminus custom indicators

For the 11.0-migration API, inject `HealthIndicatorService`, create a result
builder with `check(key)`, and return `up()` or `down()`:

```typescript
const indicator = this.healthIndicatorService.check(key);

try {
  const healthy = await this.probe();
  return healthy
    ? indicator.up()
    : indicator.down({ reason: 'probe failed' });
} catch {
  return indicator.down('Unable to run probe');
}
```

Both result methods accept optional details. `HealthIndicator` and
`HealthCheckError` are deprecated and scheduled for removal in the next major
release; new custom indicators should not derive their result flow from those
classes.

## Operations checklist

- Observe transport state through `status`.
- Subscribe to driver events through `on`.
- Call `unwrap` only when native-driver access is needed.
- Resolve microservice options through providers when configuration is injected.
- Assign NATS queues at handler granularity where routing requires it.
- Opt into the NATS graceful-shutdown path when orderly teardown matters.
- Use TCP ephemeral ports and buffer bounds where appropriate.
- Configure topic-based RMQ routing with RabbitMQ topic exchanges.
- Retain WebSocket causes and install a custom `ws` parser when needed.
- Enable JSON `ConsoleLogger` output for structured log consumers.
- Build Terminus results with `HealthIndicatorService` and migrate deprecated
  indicator classes.

