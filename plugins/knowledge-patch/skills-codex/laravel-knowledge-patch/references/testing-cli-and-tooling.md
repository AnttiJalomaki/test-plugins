# Testing, CLI, and Tooling

Test isolation and assertions, fakes, Artisan commands, generators, and development tooling.

Batch identifiers in section headings provide exact source attribution.

## Artisan failure and silence behavior (2025-12)

`cache:clear` now returns a failure exit code when clearing fails, and `queue:work` respects the standard `--quiet` and `--silent` output flags.

## Binary file response assertions (2026-03-laravel-12)

HTTP tests can use response assertions with `BinaryFileResponse` instances, so file-download responses no longer need to bypass the normal assertion flow.

## Cached bootstrap testing traits (2025-11)

The `WithCachedRoutes` and `WithCachedConfig` testing traits let application tests exercise bootstrapping with route and configuration caches; cached-configuration tests also support parallel runs.

```php
final class CachedBootstrapTest extends TestCase
{
    use WithCachedConfig, WithCachedRoutes;
}
```

## Callback-scoped queue fakes (2025-07)

`Queue::fakeFor()` and `Queue::fakeExceptFor()` temporarily scope queue faking to a callback, including a variant that lets selected jobs continue to use the real queue.

## Closure locations in route listings (2026-03-laravel-12)

`route:list` displays the source file path and line number for closure routes, making those routes traceable from CLI output.

## Config file generation (2025-08)

The new `make:config` Artisan command generates a configuration file in the application's `config` directory.

```shell
php artisan make:config services
```

## Custom command discovery (2025-12)

Console command discovery exposes a `commandFileFinder` hook and excludes test files. The follow-up behavior still discovers a real `Command` class whose name ends in `Test`.

## Development command registry (2026-06)

Laravel 13 adds `dev` and `dev:list` Artisan commands. Registered development commands track their source, support priority ordering, and the development runner stops the other commands when one fails.

```shell
php artisan dev
php artisan dev:list
```

## Direct factory inserts (2025-11)

Model factories now provide `insert()` for persisting generated rows. Hidden attributes and array-cast attributes are handled by this insertion path.

```php
User::factory()->count(100)->insert();
```

## Disabling factory parent expansion globally (2025-07)

`Factory::dontExpandRelationshipsByDefault()` globally prevents model factories from automatically expanding parent relationship definitions.

```php
Factory::dontExpandRelationshipsByDefault();
```

## Environment data in schedule listings (2025-11)

The JSON output from `schedule:list` now includes environment information for scheduled events.

## Exact queue dispatch assertions (2026-01)

`QueueFake::assertPushedTimes()` is now public, allowing tests to assert an exact job dispatch count.

```php
Queue::assertPushedTimes(SendReport::class, 2);
```

## Factory sequence context (2025-09)

Factory `Sequence` callbacks now receive the pending `$attributes` and `$parent` arguments, allowing each sequence value to depend on the model data and parent being created.

## Fake DNS validation (2026-07)

DNS lookups performed by validation rules can be faked in tests.

## Filtering schedule listings by environment (2026-05)

`schedule:list` can filter scheduled events by environment, so deployment tooling can inspect only events active for a target environment.

## Iterable database-empty assertions (2026-07)

`assertDatabaseEmpty()` accepts an iterable, allowing one assertion to verify multiple database targets.

## JSON event listings (2025-04)

The event list command can emit machine-readable output with `php artisan event:list --json`.

## Machine-readable failed-job listings (2026-05)

`queue:failed` supports JSON output, and its normal listing reports the actual job class name.

## Middleware-filtered route listings (2025-11)

`route:list` accepts a `--middleware` filter for narrowing its output to routes using a given middleware.

```shell
php artisan route:list --middleware=auth
```

## More prohibitable commands (2026-06)

`cache:clear` and `queue:flush` now participate in Laravel's command-prohibition mechanism.

## Native MariaDB CLI integration (12.0.0)

Database CLI operations for MariaDB now use native MariaDB commands, so environments invoking those operations must provide the corresponding binaries.

## Optional Faker dependency (2025-09)

Laravel can run without `fakerphp/faker` installed. Applications that do not use factories or generated fake data no longer need to carry Faker as a dependency.

## Parallel-test cache isolation (2026-02)

Parallel tests now receive process-isolated cache prefixes by default, with an option to disable the safe prefix when shared-cache behavior is intentional.

## Parallel-test maintenance state (2026-06)

An `array` maintenance-mode driver is available for parallel testing.

## Parallel-test pre-migration setup (2025-12)

Parallel database testing has a pre-migration hook, allowing database preparation to run after a test database is selected but before its migrations execute.

## PHPUnit 12.2 support (2025-06)

Laravel 12 supports PHPUnit 12.2, allowing application test dependency constraints to move beyond PHPUnit 11.

## PHPUnit 12.4 support (2025-10)

Laravel 12 supports PHPUnit 12.4, allowing application test dependency constraints to move beyond the previously supported 12.2 release.

## Prohibitable schema dumps (2025-11)

The database schema dump command can now be blocked by Laravel's destructive-command prohibition.

## Prohibiting additional Artisan commands (2026-05)

`queue:clear` and `key:generate` now participate in Laravel's command-prohibition mechanism, allowing production safety policies to block them.

## Queue-fake inspection and push hooks (2026-07)

`QueueFake` can inspect delayed and reserved jobs, exposes `beforePushing()` and `afterPushing()` hooks, and implements `creationTimeOfOldestPendingJob()` for backend-free queue tests.

## Recording non-faked HTTP requests (2025-03)

The HTTP client can record real requests without faking their responses, so tests and diagnostics can inspect traffic while it is still sent normally.

```php
Http::record();
Http::get('https://example.test');
$recorded = Http::recorded();
```

## Redis command failure listeners (2026-01)

Redis connections expose `listenForFailures()` and dispatch `CommandFailed`, allowing applications to observe failed Redis commands explicitly.

## Schema dumps without migration data (2026-06)

The schema dump command accepts `--without-migration-data` to omit migration data from the dump.

```shell
php artisan schema:dump --without-migration-data
```

## Stream bodies in HTTP fakes (2026-07)

HTTP fake responses accept stream bodies, allowing tests to model streamed response content.

## Suppressing factory callbacks (2026-02)

Factories provide `withoutAfterMaking()` and `withoutAfterCreating()` to bypass their respective lifecycle callbacks for a factory operation.

## Test-scoped string factories (13.0-upgrade)

Custom UUID, ULID, and random string factories registered through `Str` are reset during test teardown; configure them in each applicable test or setup hook.

## Test-time isolation (2026-07)

Fake time is reset globally after each test, preventing time state from leaking into later tests.

## Updated test and component dependencies (2025-12)

Laravel 12 supports PHPUnit 12.5. Reflection facilities have also been split from `illuminate/support` into a dedicated Illuminate Reflection component.

## Waiting on fake processes (2025-10)

`FakeInvokedProcess` now provides `waitUntil()`, allowing process-fake tests to cover code that waits for a condition using the same API as an invoked process.
