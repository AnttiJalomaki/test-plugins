# Modules, Configuration, and Services

Use this reference for container identity, metadata reflection, module exports,
application contexts, caching, configuration, exceptions, and health checks. It
incorporates the `11.0-migration` and `11.0.0` guidance.

## Dynamic-module identity

Dynamic-module equivalence is based on object identity rather than a
predictable hash of module metadata. Two factory calls that produce deeply
equal metadata still produce distinct dynamic-module objects.

Create the dynamic-module object once and reuse it when imports should
deduplicate:

```typescript
const usersFeature = TypeOrmModule.forFeature([User]);

await Test.createTestingModule({
  imports: [usersFeature],
}).compile();
```

When a test intentionally needs the old metadata-based behavior, select the
appropriate parent, retrieve every instance, or opt back into deep hashing:

```typescript
const usersFeature = TypeOrmModule.forFeature([User]);

await Test.createTestingModule(
  { imports: [usersFeature] },
  { moduleIdGeneratorAlgorithm: 'deep-hash' },
).compile();
```

Prefer reference reuse in application code. Treat deep hashing as an explicit
test compatibility choice.

## Reflector return values and types

For a single object-valued metadata entry, `Reflector.getAllAndMerge()` returns
the object itself instead of an array containing the object. Code that always
indexes or iterates the result as an array must be updated.

`Reflector.getAllAndOverride()` returns `T | undefined`, so consumers must
handle missing metadata. Transformed `ReflectableDecorator` types are inferred
across Reflector methods; preserve that inferred type instead of adding a
broader manual assertion.

## Module exports

Promise values are not supported in a module's `exports` list. Export resolved
providers or module tokens rather than promises. Move asynchronous construction
into the appropriate provider or dynamic-module setup.

## Selected application contexts

`NestApplicationContext.select()` can override `abortOnError` for the context it
selects:

```typescript
const featureContext = app.select(FeatureModule, {
  abortOnError: false,
});
```

Use the override when the selected feature context needs different error
termination behavior from its parent context.

## Intrinsic exceptions

`IntrinsicException` marks exceptions that Nest should not automatically log.
Use it for expected framework-level failures when automatic logging would
duplicate or add unwanted log output.

## Cache stores and raw data

The updated `@nestjs/cache-manager` expects external backends as Keyv adapters
inside a `stores` array. The former `store` configuration is not the migration
target:

```typescript
CacheModule.registerAsync({
  useFactory: async () => ({
    stores: [new KeyvRedis('redis://localhost:6379')],
  }),
});
```

Raw cached data has `{ value, expires }` shape. This affects:

- Code that consumes the backend directly rather than through the cache
  abstraction.
- Migration of entries written by an older cache configuration.
- Diagnostics or scripts that inspect serialized cache records.

Do not assume direct backend reads return only the application value.

## Configuration precedence

In `@nestjs/config@4`, `ConfigService#get()` resolves values in this order:

1. Internal configuration.
2. Validated environment values.
3. `process.env`.

Internal configuration can therefore override environment variables. Test
colliding keys when an application previously assumed that `process.env`
always won.

`ignoreEnvVars` is deprecated. Its replacement depends on the desired behavior:

- `validatePredefined: false` skips validation for environment variables that
  were present before the configuration module was imported.
- `skipProcessEnv: true` prevents `ConfigService#get()` from consulting
  `process.env`.

```typescript
ConfigModule.forRoot({
  validatePredefined: false,
  skipProcessEnv: true,
});
```

These flags solve different problems and should not be treated as synonyms.

## Terminus health indicators

Custom Terminus checks can inject `HealthIndicatorService`, start a result with
`check(key)`, and return `up()` or `down()` with optional details:

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

The former `HealthIndicator` and `HealthCheckError` classes are deprecated and
scheduled for removal in the next major release. New and migrated indicators
should use `HealthIndicatorService`.

