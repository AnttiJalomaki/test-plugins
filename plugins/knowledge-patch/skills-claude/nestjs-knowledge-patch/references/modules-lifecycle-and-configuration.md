# Modules, Lifecycle, and Configuration

## Dynamic-module identity

Under the 11.0-migration behavior, Nest determines dynamic-module equivalence by
object identity rather than by a predictable hash of module metadata. Two calls
that produce structurally equal dynamic-module objects are not automatically the
same module.

Reuse a single object when imports are intended to deduplicate:

```typescript
const usersFeature = TypeOrmModule.forFeature([User]);

@Module({ imports: [usersFeature] })
export class UsersModule {}
```

Tests that deliberately depend on the older deep-comparison behavior can opt
back into deep hashing:

```typescript
await Test.createTestingModule(
  { imports: [usersFeature] },
  { moduleIdGeneratorAlgorithm: 'deep-hash' },
).compile();
```

When multiple instances are intentional, a test can instead select the correct
parent context or retrieve every instance.

## Module exports

Promise values in a module's `exports` list are unsupported as of 11.0.0.
Resolve the provider first and export a provider or module token rather than a
promise.

## Reflector results and types

The 11.0-migration includes three related Reflector changes:

- When one metadata entry contains an object, `getAllAndMerge()` returns that
  object rather than an array containing the object.
- `getAllAndOverride()` returns `T | undefined`.
- A transformed `ReflectableDecorator` has its transformed type inferred across
  the Reflector methods.

Update array assumptions, account for `undefined`, and allow the inferred
transformed type to replace unnecessary casts.

## Lifecycle shutdown order

Termination hooks now run in the reverse of initialization order. Given
dependencies `A -> B -> C`, initialization runs `C -> B -> A`, while all three
termination phases run `A -> B -> C`:

1. `OnModuleDestroy`
2. `BeforeApplicationShutdown`
3. `OnApplicationShutdown`

Global modules initialize first and are destroyed last. This is part of the
11.0-migration guidance.

Review cleanup code that assumes a dependency was destroyed earlier. In the
example above, `A` starts terminating while `B` and `C` are still available;
`C` terminates last.

## Cache stores and raw records

The updated `@nestjs/cache-manager` integration in 11.0-migration expects
external backends as Keyv adapters in a `stores` array. Replace the older
singular `store` configuration:

```typescript
CacheModule.registerAsync({
  useFactory: async () => ({
    stores: [new KeyvRedis('redis://localhost:6379')],
  }),
});
```

Raw cached data has the shape `{ value, expires }`. This is observable to code
that accesses the backend directly and must be considered when migrating data
written in the previous format.

## Configuration lookup and validation

With `@nestjs/config@4` in 11.0-migration, `ConfigService#get()` resolves values
in this order:

1. Internal configuration.
2. Validated environment values.
3. `process.env`.

Internal configuration can consequently override a same-named environment
variable.

`ignoreEnvVars` is deprecated. Choose its replacement based on the required
behavior:

```typescript
ConfigModule.forRoot({
  validatePredefined: false,
  skipProcessEnv: true,
});
```

- `validatePredefined: false` skips validation of variables that were present
  before the module was imported.
- `skipProcessEnv: true` prevents `ConfigService#get()` from consulting
  `process.env`.

These switches are independent; enable only the behavior the application needs.

## Application-context selection

As of 11.0.0, `NestApplicationContext.select()` can override `abortOnError` for
the selected context:

```typescript
const featureContext = app.select(FeatureModule, {
  abortOnError: false,
});
```

This lets the selected context use an error-abort policy different from the
surrounding context.

## Intrinsic exceptions

`IntrinsicException` marks an exception that Nest should not automatically log
as of 11.0.0. Use it for expected framework-level failures when automatic
logging would create duplicate or unwanted output.

## Module migration checklist

- Reuse one dynamic-module object for every import that should deduplicate.
- Use deep hashing only in tests that intentionally need metadata equivalence.
- Remove promise values from module exports.
- Update Reflector result shapes and optional return types.
- Verify assumptions about termination ordering and global-module teardown.
- Configure Keyv adapters through `stores` and migrate raw cache records.
- Reassess configuration collisions under internal-first lookup precedence.
- Replace `ignoreEnvVars` with the option matching the intended behavior.
- Set `abortOnError` at context selection when the selected context differs.
- Mark only expected, non-automatically-logged failures as intrinsic.

