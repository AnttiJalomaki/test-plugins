---
name: laravel-knowledge-patch
description: Laravel
version: 13.23.0
license: MIT
metadata:
  author: Nevaberry
---


# Laravel Knowledge Patch

Use this skill when upgrading or maintaining a Laravel application, package,
queue driver, cache store, database integration, or contract implementation.

Inspect `composer.json` and the lockfile before applying version-dependent
behavior. Prefer application code, tests, configuration, and installed behavior.

## Reference Index

| Reference | Topics |
| --- | --- |
| [Framework core, container, and utilities](references/framework-core-and-container.md) | Runtime requirements, dependencies, contracts, container behavior, collections, strings, enums, and helpers |
| [Database, Eloquent, and schema](references/database-eloquent-and-schema.md) | Queries, models, relationships, casts, migrations, schema operations, and database drivers |
| [Queues, jobs, and scheduling](references/queues-jobs-and-scheduling.md) | Queue routing, job attributes, workers, metrics, batches, listeners, and schedules |
| [HTTP, routing, requests, and processes](references/http-routing-and-processes.md) | HTTP clients, route matching, middleware, requests, responses, URIs, maintenance mode, and processes |
| [Cache, sessions, Redis, and filesystems](references/cache-session-and-filesystem.md) | Cache contracts, serialization, locks, sessions, Redis, disks, uploads, and filesystem behavior |
| [Validation, authentication, and security](references/validation-auth-and-security.md) | Validation rules, password handling, authentication, encryption, and request protection |
| [Testing, CLI, and tooling](references/testing-cli-and-tooling.md) | Assertions, fakes, test isolation, Artisan commands, generators, and development tooling |
| [Mail, notifications, and broadcasting](references/mail-notifications-and-broadcasting.md) | Mail transports, notification lifecycle, attachments, broadcasting, and delivery integrations |
| [Views, resources, and frontend integration](references/views-resources-and-frontend.md) | Blade, views, API resources, pagination, Vite, Markdown, and JSON presentation |
| [Events, observability, and operations](references/events-observability-and-operations.md) | Events, logging, exception reporting, health behavior, and runtime telemetry |

## Upgrade Priorities

For a Laravel 13 migration, resolve these changes before adopting optional
features:

1. Run PHP 8.3 or newer and update Composer constraints for
   `laravel/framework:^13.0` and `laravel/tinker:^3.0`.
2. Update optional tooling to `laravel/boost:^2.0`, PHPUnit 12, or Pest 4 as
   applicable.
3. Audit cache prefixes, Redis prefixes, session cookie names, serialized
   cache payloads, and custom cache stores.
4. Review custom framework contracts, HTTP response subclasses, queue
   drivers, middleware references, and manager extensions.
5. Exercise overlapping domain routes, Eloquent boot methods, polymorphic
   pivot models, MySQL upserts, and joined deletes.
6. Update exact rendering, mail, queue-event, and test assertions.

Laravel Boost 2 can guide an installed Laravel 12 application through the
upgrade:

```text
/upgrade-laravel-v13
```

Keep integration coverage around database writes, queue serialization, route
matching, cache payloads, request protection, and application boot. Those
areas include behavior changes that dependency updates alone cannot verify.

## Runtime and Dependency Changes

Laravel 13 requires PHP 8.3 and supports PHP 8.3 through 8.5. Laravel 12
supports PHP 8.2 through 8.5.

Remove legacy global helpers that conflict with `symfony/polyfill-php85`.
Below PHP 8.5 the polyfill may define names such as `array_first()` and
`array_last()`. Use `Arr::first()` and related helpers when callback behavior
is required.

Review optional queue dependencies during the upgrade. The Beanstalkd
integration supports `pda/pheanstalk` 8.x and no longer supports 5.x.

## Cache and Session Compatibility

Framework fallback cache prefixes, Redis prefixes, and session cookie names
use hyphenated slugs with `-cache-`, `-database-`, and `-session` suffixes.
Set these explicitly to retain existing identifiers:

```dotenv
CACHE_PREFIX=example-cache-
REDIS_PREFIX=example-database-
SESSION_COOKIE=example-session
```

The default `cache.serializable_classes` value is `false`. Allow-list cached
PHP object classes or migrate cached values to non-object payloads:

```php
'serializable_classes' => [
    App\Data\CachedDashboardStats::class,
],
```

Custom cache stores must implement TTL extension through
`touch($key, $seconds)`. Cache locks can also refresh their expiration for
long-running work.

## Container and Contract Changes

Constructor resolution and `Container::call()` honor nullable class
parameters with defaults. A parameter such as `?Carbon $date = null` receives
`null` when no explicit binding exists.

Custom implementations must account for added framework contract methods,
including:

- `Dispatcher::dispatchAfterResponse($command, $handler = null)`
- the current `ResponseFactory::eventStream` signature
- `MustVerifyEmail::markEmailAsUnverified()`
- queue metrics such as `pendingSize`, `delayedSize`, `reservedSize`, and
  `creationTimeOfOldestPendingJob`

Manager `extend()` callbacks are bound to the manager. Capture another object
with `use (...)` instead of assuming it remains available as `$this`.

Use `#[BindWhen]` for conditional container bindings. Contextual attributes
can inspect their target reflection parameter, allowing parameter-specific
resolution.

## Database and Eloquent Breakpoints

MySQL and MariaDB `upsert()` require a non-empty `uniqueBy` argument. Joined
MySQL deletes retain requested `ORDER BY` and `LIMIT` clauses; incompatible
database variants raise `QueryException` instead of running an unbounded
delete.

Do not instantiate a model while that same model is still executing its
`boot` or trait `boot*` methods. Laravel now raises `LogicException`; move the
nested construction outside the boot cycle.

Inferred table names for polymorphic pivot models using custom pivot classes
are pluralized. Set `$table` explicitly when retaining an older singular
name.

Serialized Eloquent collections restore eager-loaded relations along with
their models. Recheck queued payload size and code that previously expected
relations to be absent after restoration.

For schema work:

- MariaDB supports vector indexes.
- PostgreSQL connections support transaction poolers.
- PostgreSQL column changes may use `->using(...)->change()` for conversion
  expressions.
- SQLite connections accept `file:` URI database names.

## Routing and Request Protection

Explicit-domain routes match before routes without domains, regardless of
registration order. Test overlapping domain and non-domain definitions.

The request-forgery middleware is `PreventRequestForgery` and validates the
request origin through `Sec-Fetch-Site`. Replace direct references to the
deprecated `VerifyCsrfToken` and `ValidateCsrfToken` aliases, and use
`preventRequestForgery(...)` in application configuration.

Routing unserialization restricts the classes it may instantiate. Avoid
arbitrary custom objects in serialized route values.

Routes can carry metadata, and `RouteParameter` can infer its route key from
the attributed parameter name. Controllers may declare exclusions with
`#[WithoutMiddleware]`.

## Queues and Scheduling

Centralize job destinations with `Queue::route()`:

```php
Queue::route(ProcessPodcast::class, connection: 'redis', queue: 'podcasts');
```

A single string passed to `Queue::route()` is the queue name. Class-level
queue attributes include `#[Tries]`, `#[Backoff]`, `#[Timeout]`,
`#[FailOnTimeout]`, and `#[Delay]`; runtime `onQueue()` selection takes
precedence.

Queue event consumers must use `QueueBusy::$connectionName`.
`JobAttempted::$exception` contains an exception or `null` instead of exposing
the old boolean occurrence flag.

The scheduler has dedicated pause and resume commands:

```shell
php artisan schedule:pause
php artisan schedule:resume
```

Schedules supplied through `ApplicationBuilder::withScheduling()` are
registered when `Schedule` is resolved. Do not rely on immediate registration
during bootstrap.

Use queue inspection APIs instead of reaching into a backend directly.
`InspectedJob` includes payload and queue information. Worker stopping events
also expose processed-job count, last-job time, and memory usage.

## HTTP and Process Behavior

Pools created from `PendingRequest` default to concurrency two. Specify a
different value explicitly when throughput or upstream limits require it.

Laravel's HTTP client can act directly as a PSR client. HTTP fakes accept
stream bodies, while `Http::query()` and the testing `query()` / `queryJson()`
helpers expose request query data.

Pending process timeouts and retry sleeps accept `CarbonInterval` values:

```php
Process::timeout(CarbonInterval::seconds(30))
    ->run('php artisan report:build');
```

Malformed cursor decoding returns `null`; null-check `Cursor::fromEncoded()`
when the value came from an untrusted request.

## Attributes and Application Features

Controller middleware and authorization checks may be colocated with
`#[Middleware]` and `#[Authorize]`. Framework attributes declared on supported
parents or traits are inherited, while a child model's `Table` attribute
overrides its parent's.

Laravel's provider-independent AI SDK supports agents, tools, embeddings,
image and audio generation, and vector stores. The query builder can run
semantic searches through PostgreSQL and `pgvector`:

```php
$documents = DB::table('documents')
    ->whereVectorSimilarTo('embedding', 'Best wineries in Napa Valley')
    ->limit(10)
    ->get();
```

First-party image processing is available, and
`ImageManager::fromStorage()` accepts enum disk selectors.

## Rendering and Test Adjustments

`Js::from()` emits unescaped Unicode by default. Update assertions that expect
`\u` escape sequences.

Direct Bootstrap 3 pagination view references are
`pagination::bootstrap-3` and `pagination::simple-bootstrap-3`.

Custom UUID, ULID, and random string factories registered through `Str` reset
during test teardown. Register them in each applicable setup hook. Fake time
also resets globally after every test.

Use the linked references for the complete behavior set and exact attribution.
