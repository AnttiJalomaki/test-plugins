# Events, Observability, and Operations

Lifecycle events, logging, exception reporting, health behavior, and runtime telemetry.

Batch identifiers in section headings provide exact source attribution.

## Callback-based exception report suppression (2025-07)

`dontReportUsing()` registers a callback that filters exceptions from reporting when a class-only `dontReport` list is not expressive enough.

## Lazy exceptions in `throw_if()` (2025-10)

`throw_if()` accepts a closure for lazily constructing the exception when its condition is true.

```php
throw_if($invalid, fn () => new DomainException('Invalid state'));
```

## Monthly log rotation (2026-07)

Laravel 13 includes a monthly log driver and a corresponding logging channel configuration.

## Named event arguments (2026-02)

Event classes can be dispatched or broadcast with named constructor arguments, so callers no longer have to supply every event argument positionally.

```php
OrderShipped::dispatch(order: $order, notify: true);
```

## Number parsing failures (2025-09)

The locale-aware number parsing helpers may return `false`; callers of `Number::parseInt()` and `Number::parseFloat()` must account for parse failure rather than assuming a numeric result.

## Reloadable service lifecycle (2025-12)

Laravel now provides a reload command and lets services register for reloading. Queue workers may also opt out of the `queuePaused` and `queueShouldRestart` cache checks when those controls are not needed.

## Selective log-context removal (2025-03)

`Log::withoutContext()` accepts keys to remove only selected values from subsequent log context.

```php
Log::withoutContext(['tenant_id', 'trace_id']);
```

## Server-provided application base paths (2025-09)

Application bootstrap may read `APP_BASE_PATH` from `$_SERVER`, allowing a host or bootstrap wrapper to set the base path before the application is loaded.

```php
$_SERVER['APP_BASE_PATH'] = '/srv/application';
```

## Structured JSON logging (2026-04)

Laravel 13 introduces `JsonFormatter` for JSON log output, including exception context when the exception handler is not bound.
