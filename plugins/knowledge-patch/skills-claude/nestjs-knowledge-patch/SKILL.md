---
name: nestjs-knowledge-patch
description: NestJS
version: 11.1.x selected changes
license: MIT
metadata:
  author: Nevaberry
---


# NestJS Knowledge Patch

Use this skill when maintaining or migrating NestJS applications whose code may
depend on current platform adapters, dependency injection, lifecycle behavior,
configuration, health checks, logging, or transport APIs.

Prefer the application's manifest, code, and tests when they disagree with this
guidance. Check the installed versions of Nest core and companion packages before
applying package-specific migration advice.

## Reference index

| Reference | Topics |
| --- | --- |
| [Platform and routing](references/platform-and-routing.md) | Runtime and Express requirements, Fastify CORS and path matching, middleware order, WebDAV methods, date parsing |
| [Modules, lifecycle, and configuration](references/modules-lifecycle-and-configuration.md) | Dynamic-module identity, Reflector types, shutdown order, promise exports, cache stores, config precedence, context selection, intrinsic exceptions |
| [Transports and operations](references/transports-and-operations.md) | Microservice state and native access, NATS, TCP, RabbitMQ, WebSockets, JSON logs, Terminus indicators |

## First-pass migration checks

1. Confirm the runtime is Node.js 20 or newer.
2. Treat the default Express adapter as Express 5 code.
3. If using Fastify, list every required non-safelisted CORS method explicitly.
4. Replace middleware `(.*)` patterns with named wildcards such as `*splat`.
5. Reuse dynamic-module objects when imports are intended to represent one module.
6. Remove promises from module `exports` arrays.
7. Recheck assumptions about metadata merge return shapes and override types.
8. Verify shutdown-hook and global-middleware ordering in order-sensitive code.
9. Migrate cache backends to Keyv adapters in `stores`.
10. Re-evaluate configuration precedence and deprecated environment options.
11. Replace deprecated Terminus indicator base classes with
    `HealthIndicatorService`.

## Breaking platform changes

### Runtime and Express

The supported runtime starts at Node.js 20. Node.js 16 and 18 are not supported.
The default Express platform integration uses Express 5.

### Fastify CORS

Fastify's default CORS methods are safelisted methods. Applications that accept
`PUT`, `PATCH`, `DELETE`, or other non-safelisted methods must enable them:

```typescript
app.enableCors({ methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'] });
```

### Middleware path syntax

Middleware matching uses the current `path-to-regexp` syntax. Replace anonymous
`(.*)` matches with a named wildcard:

```typescript
consumer.apply(ApiMiddleware).forRoutes('*splat');
```

This migration applies to Nest middleware matching. Do not rewrite ordinary
Fastify route wildcards solely because of this change.

## Dependency injection and metadata

### Dynamic-module identity

Dynamic modules are equivalent only when the same object is reused. Store a
dynamic-module result and import that reference everywhere it should be
deduplicated:

```typescript
const usersFeature = TypeOrmModule.forFeature([User]);

@Module({ imports: [usersFeature] })
export class UsersModule {}
```

For tests that intentionally need metadata-based equivalence, opt into deep
hashing:

```typescript
await Test.createTestingModule(
  { imports: [usersFeature] },
  { moduleIdGeneratorAlgorithm: 'deep-hash' },
).compile();
```

Tests can instead select the intended parent or retrieve all module instances
when multiple instances are expected.

### Reflector return values

When a single metadata entry contains an object, `getAllAndMerge()` returns the
object itself, not a one-element array. `getAllAndOverride()` returns
`T | undefined`. Transformed `ReflectableDecorator` types now flow through the
Reflector APIs, so remove stale casts and update affected assertions.

### Promise exports

Module `exports` arrays cannot contain promises. Export resolved providers or
module tokens.

## Ordering changes

### Shutdown hooks

Termination hooks run in reverse initialization order. For dependencies
`A -> B -> C`, initialization is `C -> B -> A`, while
`OnModuleDestroy`, `BeforeApplicationShutdown`, and
`OnApplicationShutdown` run `A -> B -> C`.

Global modules initialize first and are destroyed last. Audit cleanup code when
one provider assumes that a dependency has already shut down.

### Global middleware

Middleware registered by global modules runs before middleware registered by
imported modules, regardless of where the global module appears in the dependency
graph.

## Cache and configuration

### Keyv-backed caches

Configure external cache backends as Keyv adapters in `stores`:

```typescript
CacheModule.registerAsync({
  useFactory: async () => ({
    stores: [new KeyvRedis('redis://localhost:6379')],
  }),
});
```

Direct backend consumers must handle raw cache records shaped as
`{ value, expires }`. Account for that shape when reading or migrating old data.

### Configuration precedence

`ConfigService#get()` resolves internal configuration first, then validated
environment values, then `process.env`. Internal configuration can therefore
override an environment variable with the same key.

Replace deprecated `ignoreEnvVars` according to intent:

```typescript
ConfigModule.forRoot({
  validatePredefined: false,
  skipProcessEnv: true,
});
```

Use `validatePredefined: false` to avoid validating variables that existed before
module import. Use `skipProcessEnv: true` to prevent `get()` from consulting
`process.env`.

## Health checks

Inject `HealthIndicatorService`, begin with `check(key)`, and return `up()` or
`down()` with optional details:

```typescript
const indicator = this.healthIndicatorService.check(key);

try {
  const healthy = await this.probe();
  return healthy ? indicator.up() : indicator.down({ reason: 'probe failed' });
} catch {
  return indicator.down('Unable to run probe');
}
```

`HealthIndicator` and `HealthCheckError` are deprecated and scheduled for removal
in the next major release.

## Common application features

### Structured console logs

Enable JSON output by supplying a configured `ConsoleLogger`:

```typescript
const app = await NestFactory.create(AppModule, {
  logger: new ConsoleLogger({ json: true }),
});
```

### Date query parameters

Use the exported `ParseDatePipe` to transform an incoming value to `Date`:

```typescript
find(@Query('since', ParseDatePipe) since: Date) {}
```

### Selected contexts and expected exceptions

`NestApplicationContext.select()` accepts a context-specific `abortOnError`:

```typescript
const featureContext = app.select(FeatureModule, { abortOnError: false });
```

Use `IntrinsicException` for expected framework-level exceptions that Nest should
not log automatically, avoiding duplicate or unwanted log output.

## Transport quick reference

- Microservice clients and servers expose `status`, `on`, and `unwrap` for state,
  native events, and native-driver access.
- NATS handlers can select queues individually; the transporter can shut down
  gracefully.
- TCP supports operating-system-selected ports and a maximum packet-buffer size.
- Microservice options can be resolved through dependency injection.
- RabbitMQ supports topic exchanges.
- WebSocket errors may retain a cause, and the `ws` adapter can use a custom
  message parser.
- WebDAV methods are recognized by the common, core, and Fastify packages.

Use the transport reference for the full behavior and migration implications.

