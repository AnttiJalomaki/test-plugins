---
name: nestjs-knowledge-patch
description: NestJS
version: 11.1.x selected changes
license: MIT
metadata:
  author: Nevaberry
---


# NestJS Knowledge Patch

Use this skill when writing, migrating, reviewing, or debugging a NestJS
application, library, adapter, or microservice. Inspect the installed
`@nestjs/*` packages and the selected HTTP or transport adapter before applying
the guidance.

## Reference index

| Reference | Topics |
| --- | --- |
| [Platform, HTTP, and lifecycle](references/platform-http-and-lifecycle.md) | Runtime requirements, Express and Fastify migrations, middleware, shutdown hooks, WebDAV, WebSockets, and date parsing |
| [Modules, configuration, and services](references/modules-configuration-and-services.md) | Dynamic modules, reflection, exports, context selection, exceptions, caching, configuration, and health indicators |
| [Microservices and observability](references/microservices-and-observability.md) | Transport status and native access, JSON logs, NATS, TCP, dependency-injected options, and RabbitMQ |

## Working method

1. Read `package.json` and the lockfile to identify the core, platform,
   cache-manager, config, Terminus, and microservice package versions actually
   installed.
2. Identify whether the application uses Express, Fastify, `ws`, NATS, TCP,
   RabbitMQ, or a custom transport.
3. For a migration, address runtime and removed behavior before adopting new
   APIs.
4. Test lifecycle order, middleware order, configuration precedence, and
   external-cache data explicitly; these changes can be valid at compile time
   while altering runtime behavior.
5. Open the focused reference file for the complete constraints and migration
   notes.

## Breaking changes and deprecations

### Require Node.js 20 or newer

NestJS 11 no longer supports Node.js 16 or 18. Update local development, CI,
containers, and deployment runtimes together. The default Express integration
uses Express 5, so exercise routing and middleware behavior during the upgrade.

### Configure Fastify CORS methods

With `@nestjs/platform-fastify` v11 and Fastify v5, the default CORS methods are
only the safelisted methods. Explicitly add application methods such as `PUT`,
`PATCH`, and `DELETE`:

```typescript
app.enableCors({
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
});
```

### Migrate middleware wildcards

Nest middleware matching uses the latest `path-to-regexp`. Replace an anonymous
`(.*)` middleware matcher with a named wildcard:

```typescript
consumer.apply(ApiMiddleware).forRoutes('*splat');
```

Do not mechanically change ordinary Fastify route wildcards; this migration
applies to Nest middleware matching.

### Reuse dynamic-module objects

Dynamic-module equivalence is based on reference identity. Calling a dynamic
module factory twice creates distinct module definitions even when their
metadata is deeply equal. Create the definition once and reuse it when the
imports must deduplicate:

```typescript
const usersFeature = TypeOrmModule.forFeature([User]);

@Module({
  imports: [usersFeature],
})
export class AppModule {}
```

Tests that specifically rely on metadata hashing can opt into
`moduleIdGeneratorAlgorithm: 'deep-hash'`. See the modules reference for the
other ways to disambiguate repeated instances.

### Update Reflector result assumptions

For one object-valued metadata entry, `Reflector.getAllAndMerge()` returns the
object itself, not a one-element array. Remove array-only assumptions.
`getAllAndOverride()` returns `T | undefined`; handle the missing-metadata case.
Transformed `ReflectableDecorator` types now flow through Reflector method
inference.

### Remove promise values from module exports

A module `exports` list cannot contain promises. Await construction elsewhere
and export a resolved provider or module token.

### Verify shutdown and middleware order

Termination hooks run in reverse initialization order. For dependencies
`A -> B -> C`, initialization is `C -> B -> A`, while
`OnModuleDestroy`, `BeforeApplicationShutdown`, and
`OnApplicationShutdown` run `A -> B -> C`.

Global modules initialize first and are destroyed last. Their middleware also
runs before middleware from imported modules, regardless of where the global
module sits in the dependency graph.

### Migrate cache backends to Keyv adapters

Configure external backends as Keyv adapters in `stores`, not with the former
`store` option:

```typescript
CacheModule.registerAsync({
  useFactory: async () => ({
    stores: [new KeyvRedis('redis://localhost:6379')],
  }),
});
```

Raw backend entries now have `{ value, expires }` shape. Account for that shape
in direct backend consumers and when handling data written by older
configurations.

### Apply the new configuration precedence

`ConfigService#get()` resolves values in this order:

1. Internal configuration.
2. Validated environment values.
3. `process.env`.

Internal configuration can therefore override an environment variable.
`ignoreEnvVars` is deprecated. Use `validatePredefined: false` to avoid
validating variables that existed before module import, and use
`skipProcessEnv: true` to prevent `get()` from consulting `process.env`.

```typescript
ConfigModule.forRoot({
  validatePredefined: false,
  skipProcessEnv: true,
});
```

### Replace deprecated Terminus indicator classes

Custom health indicators should inject `HealthIndicatorService`, begin with
`check(key)`, and return `up()` or `down()`. `HealthIndicator` and
`HealthCheckError` are deprecated and scheduled for removal in the next major
release.

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

## High-use capabilities

### Observe and access microservice transports

Microservice client and server abstractions expose:

- `status` for observing transport state.
- `on` for subscribing to native-driver events.
- `unwrap` for accessing the underlying driver.

Keep ordinary application code on Nest abstractions and use `unwrap` only where
native-driver access is actually required.

### Emit structured console logs

Enable JSON output by providing a configured `ConsoleLogger`:

```typescript
const app = await NestFactory.create(AppModule, {
  logger: new ConsoleLogger({ json: true }),
});
```

### Parse query dates with a built-in pipe

`ParseDatePipe` is exported from `@nestjs/common` and transforms an incoming
value into a `Date`:

```typescript
find(@Query('since', ParseDatePipe) since: Date) {}
```

### Select application-context error behavior

Override `abortOnError` for a selected context:

```typescript
const featureContext = app.select(FeatureModule, {
  abortOnError: false,
});
```

### Avoid duplicate logs for intrinsic failures

Use `IntrinsicException` for expected framework-level exceptions that Nest
should not log automatically.

### Configure transports more precisely

NATS handlers can select queues individually, and the transporter supports an
optional graceful-shutdown path. TCP can use an operating-system-selected port
and can bound its maximum packet-buffer size. Microservice options may be
resolved through dependency injection. RabbitMQ supports topic exchanges.

Consult the microservices reference before configuring these behaviors, because
the exact option surface belongs to the selected transporter and driver.

### Use newer protocol extension points

WebDAV methods are recognized across the common, core, and Fastify packages.
WebSocket errors can retain a cause, and the `ws` adapter provides a
message-parser extension point for custom wire formats.

