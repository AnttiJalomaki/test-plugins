---
name: laravel-knowledge-patch
description: Laravel
version: 13.23.0
license: MIT
metadata:
  author: Nevaberry
---


# Laravel Knowledge Patch

Use this skill while upgrading, reviewing, debugging, or extending a Laravel
application. Start from the application's `composer.json`, inspect its
configuration and migrations, and load only the references needed for the
subsystem being changed.

## Working method

1. Read `composer.json` before proposing framework, PHP, PHPUnit, Pest, Symfony,
   Redis-client, Beanstalkd-client, or mail-transport constraints.
2. Determine whether the application is upgrading or already runs the target
   framework generation. Treat upgrade-only behavior as migration work, not a
   recommendation for an older application.
3. Inspect application overrides of framework contracts, queue drivers, cache
   stores, HTTP responses, middleware, model boot methods, and database
   grammars. These extension points carry the highest compatibility risk.
4. Check generated configuration and migration stubs against customized local
   copies; defaults do not retroactively update published application files.
5. Preserve driver differences. MySQL, MariaDB, PostgreSQL, SQLite, Redis,
   SQS, local disks, and managed queues do not expose identical behavior.
6. Use exact public API names and named arguments from the references. Several
   similarly named attributes, traits, properties, and options were renamed or
   removed.
7. Add focused regression tests for routing precedence, serialization, session
   rotation, queue lifecycle, validation strictness, and schema SQL whenever
   those behaviors affect the application.

## Reference index

| Reference | Load for |
| --- | --- |
| [upgrades-and-core.md](references/upgrades-and-core.md) | Framework upgrades, dependencies, contracts, bootstrap, maintenance, and core lifecycle |
| [container-auth-routing.md](references/container-auth-routing.md) | Container injection, contextual attributes, authentication, routing, middleware, context, and rate limiting |
| [database-and-schema.md](references/database-and-schema.md) | Connections, query builder, migrations, schema grammar, database CLI, and transactions |
| [eloquent-and-resources.md](references/eloquent-and-resources.md) | Models, relationships, casts, factories, scopes, serialization, and API resources |
| [cache-filesystem-redis.md](references/cache-filesystem-redis.md) | Cache, sessions, Redis, locks, password repositories, and filesystem behavior |
| [queues-concurrency-scheduling.md](references/queues-concurrency-scheduling.md) | Jobs, batches, queue drivers, workers, concurrency, schedules, and process execution |
| [http-mail-notifications.md](references/http-mail-notifications.md) | HTTP client, event streams, mail transports, mailables, notifications, and broadcasting |
| [validation-and-testing.md](references/validation-and-testing.md) | Validation rules, form requests, test fakes, assertions, cached bootstrap, and parallel tests |
| [collections-strings-support.md](references/collections-strings-support.md) | Collections, arrays, strings, numbers, URIs, JSON Schema, translation, and helper APIs |
| [blade-frontend-console-observability.md](references/blade-frontend-console-observability.md) | Blade, Vite, pagination, Artisan, event discovery, logging, and operational output |

## Upgrade quick reference

### Dependency and runtime gates

- For a Laravel 12 upgrade, require `laravel/framework:^12.0`, Carbon 3, and a
  supported PHPUnit or Pest line. Do not retain Carbon 2 assumptions.
- For a Laravel 13 upgrade, require `laravel/framework:^13.0` and
  `laravel/tinker:^3.0`; align optional Boost, PHPUnit, and Pest constraints.
- Require PHP 8.3 or newer before moving to Laravel 13.
- Recheck Pheanstalk constraints: Laravel 13 supports 8.x and drops 5.x.

### Laravel 13 breaking changes

- Implement `touch($key, $seconds)` on custom cache stores and the newly
  declared queue-size methods on custom queue drivers.
- Allow-list cached object classes under `cache.serializable_classes`, or
  replace cached PHP objects with scalar or array payloads.
- Set `CACHE_PREFIX`, `REDIS_PREFIX`, and `SESSION_COOKIE` explicitly when
  existing key and cookie identifiers must remain stable.
- Pass a non-empty `uniqueBy` to MySQL and MariaDB `upsert()`.
- Move same-model construction out of that model's `boot` and trait boot
  methods; nested construction during boot now throws.
- Rename queue listener access from `QueueBusy::$connection` to
  `$connectionName`, and read `JobAttempted::$exception` instead of
  `$exceptionOccurred`.
- Replace direct CSRF middleware references with `PreventRequestForgery` and
  configure it through `preventRequestForgery(...)`.
- Update custom HTTP response overrides to accept the callback parameters on
  `throw()` and `throwIf()`.
- Review overlapping domain and domainless routes: explicit-domain routes take
  precedence.

### Laravel 12 behavior changes

- Convert custom `DatabaseTokenRepository` expiry values from minutes to
  seconds.
- Expect associative `Concurrency::run()` input to produce keyed output.
- Preserve optional class-typed defaults: container resolution leaves a
  defaulted nullable parameter at its default value.
- Construct `Blueprint` and database `Grammar` objects with a `Connection`;
  remove `setConnection()` and `withTablePrefix()` usage.
- Expect `HasUuids` to generate ordered UUIDv7 identifiers; use
  `HasVersion4Uuids` when UUIDv4 output must remain.
- Configure the local disk root explicitly if files must stay under
  `storage/app`; the fallback root is private storage.
- Opt into SVG validation deliberately with `image:allow_svg` or
  `File::image(allowSvg: true)`.

## Database quick reference

### Schema inspection and SQL generation

```php
$tables = Schema::getTables(schema: ['main', 'blog']);
$names = Schema::getTableListing(
    schema: 'main',
    schemaQualified: false,
);
```

- Schema inspection spans all schemas by default. Do not assume unqualified
  table names.
- Use `$connection->getTablePrefix()`; prefix accessors on `Blueprint` and
  `Grammar` are deprecated.
- Verify online and instant index or column operations against the active
  database driver.
- Use explicit PostgreSQL vector, `tsvector`, full-text-vector, virtual-column,
  nulls-not-distinct, and conversion-expression APIs where applicable.
- Keep MariaDB-specific UUID and vector support separate from MySQL behavior.

### Query safety

- Treat MySQL joined deletes with `ORDER BY` or `LIMIT` as real bounded SQL;
  unsupported variants now fail instead of silently becoming unbounded.
- Expect locale number parsing to return `false` on invalid input.
- Null-check `Cursor::fromEncoded()` for malformed client cursors.
- Use strict validation rules when boolean, numeric, or integer strings must not
  be accepted.

## Queue and scheduler quick reference

```php
Queue::route(
    ProcessPodcast::class,
    connection: 'redis',
    queue: 'podcasts',
);
```

- A single string passed to `Queue::route()` names the queue, not the
  connection.
- A runtime `onQueue()` selection overrides a class-level queue attribute.
- A queue worker memory limit of zero disables memory verification.
- Release unique-job locks when an after-commit job is discarded by rollback;
  the framework handles this for standard unique jobs.
- Use pause, resume, inspection, failover, debounce, deferred execution, and
  worker lifecycle APIs instead of reaching directly into backend internals.
- For SQS fair queues, use the `messageGroup` property.
- When generating custom job-table migrations, use widened `attempts` storage.

## HTTP, security, and test quick reference

```php
Http::preventStrayRequests(allowedUrls: [
    'https://telemetry.example/*',
]);

$data = Http::get($url)->json(flags: JSON_BIGINT_AS_STRING);
```

- `Auth::login()` rotates the session identifier.
- `Request::mergeIfMissing()` treats dotted keys as nested paths.
- Replace deprecated `Request::get()` with the accessor for the intended input
  source.
- Expect the first registered duplicate route name to win in cached and
  uncached routing.
- Use `Http::record()` when requests must remain real but observable.
- Set explicit pool or batch concurrency; pending-request pools otherwise use a
  small default.
- Reject email addresses containing line breaks and enable bcrypt input-length
  enforcement where truncation would be unsafe.
- Reset framework globals, fake time, string factories, and strict form-request
  state between tests.

## High-value feature selection

- Use memoized cache stores for request-local repeated reads, including
  `flexible()` stale-while-revalidate operations.
- Use automatic relationship loading carefully to reduce accidental N+1
  queries, and cover access patterns with query-count tests.
- Use `Http::batch()` for coordinated requests and `defer()` when execution can
  move after the response.
- Use `Queue::fakeFor()` or `fakeExceptFor()` to scope faking to a callback.
- Use `Context::scope()` for temporary context and `remember()` for lazily
  initialized context values.
- Use conditional model-creation closures when values are expensive and needed
  only for insertion.
- Use queue, controller, container, model, and contextual PHP attributes when
  colocated declarative configuration is clearer than provider registration.
- Use JSON, schedule, route, event, and failed-job command output for
  automation instead of scraping tables.
