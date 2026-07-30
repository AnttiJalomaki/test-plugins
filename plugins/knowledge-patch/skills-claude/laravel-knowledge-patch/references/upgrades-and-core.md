# Upgrades and Core Lifecycle

## Dependency and platform requirements

- **Laravel 12 dependency requirements** `[12.0-upgrade]`: require
  `laravel/framework:^12.0`; use PHPUnit 11 or Pest 3 as the initial test
  baseline, and upgrade from Carbon 2 to Carbon 3.
- **Laravel 12 support window** `[12.0.0]`: PHP 8.2–8.5 is supported. Bug fixes
  run through August 13, 2026, and security fixes through February 24, 2027.
- **Laravel 12 starter kits** `[12.0.0]`: React, Svelte, and Vue kits use
  Inertia 2, TypeScript, shadcn/ui, and Tailwind; Livewire uses Flux UI and
  Volt. WorkOS AuthKit variants add social auth, passkeys, and SSO. Breeze and
  Jetstream receive no further updates.
- **Updated compatibility** `[2025-11]`: Laravel 12 accepts Resend 1.x and
  Symfony 7.4.
- **Test and component dependencies** `[2025-12]`: Laravel 12 supports PHPUnit
  12.5; Illuminate Reflection is now a separate component rather than part of
  `illuminate/support`.
- **Laravel 13 dependency requirements** `[13.0-upgrade]`: require
  `laravel/framework:^13.0` and `laravel/tinker:^3.0`; where used, align to
  `laravel/boost:^2.0`, `phpunit/phpunit:^12.0`, or `pestphp/pest:^4.0`.
- **Guided Boost upgrades** `[13.0-upgrade]`: Laravel Boost 2 can guide a
  Laravel 12 application with `/upgrade-laravel-v13`.
- **Laravel 13 support window** `[13.0.0]`: PHP 8.3–8.5 is supported. Bug fixes
  run through Q3 2027 and security fixes through March 17, 2028.
- **PHP 8.5 helper conflicts** `[13.0-upgrade]`: remove legacy global helpers
  that conflict with `symfony/polyfill-php85` definitions such as
  `array_first()` and `array_last()`; use `Arr::first()` for callback behavior.

## Framework contracts and extension points

- **Laravel 12 contract additions** `[12.0.0]`: custom implementations must add
  `ResponseFactory::streamJson()`, `CursorPaginator::hasMorePages()`,
  `Paginator::withQueryString()`, and session `flash()`.
- **Environment adapters** `[12.0.0]`: register custom environment loaders with
  `Env::extend()`.
- **Manager callback binding** `[13.0-upgrade]`: manager `extend()` closures are
  bound to the manager. Explicitly capture any other object previously assumed
  to be `$this`.
- **Laravel 13 contract additions** `[13.0-upgrade]`: custom implementations
  must add `Dispatcher::dispatchAfterResponse($command, $handler = null)`, the
  current `ResponseFactory::eventStream` signature, and
  `MustVerifyEmail::markEmailAsUnverified()`.
- **HTTP response override signatures** `[13.0-upgrade]`: custom response
  classes must accept callback parameters on `throw($callback = null)` and
  `throwIf($condition, $callback = null)`.
- **Lazy and proxy helpers** `[2025-12]`: use the support layer's first-party
  `lazy` and `proxy` object helpers.
- **First-party AI SDK** `[13.0.0]`: Laravel provides provider-independent
  agents, tools, embeddings, image and audio generation, and vector-store
  integration.
- **First-party image processing** `[2026-07]`: Laravel 13 includes image
  processing; `ImageManager::fromStorage()` accepts enum disk selectors.

## Bootstrap and application lifecycle

- **Server-provided base paths** `[2025-09]`: application bootstrap may read
  `APP_BASE_PATH` from `$_SERVER`, allowing a wrapper to set the base path
  before loading the application.
- **Reloadable services** `[2025-12]`: use the reload command and service reload
  registration. Queue workers may opt out of `queuePaused` and
  `queueShouldRestart` cache checks.
- **Scheduler-aware reloads** `[2026-02]`: reload also interrupts schedule
  execution.
- **Readable encrypted environments** `[2026-01]`: `env:encrypt --readable`
  leaves key names visible while encrypting values.
- **Maintenance mode extension** `[2025-07]`: extend maintenance backends
  through the `MaintenanceMode` facade.
- **Datetime maintenance retry** `[2026-01]`: `down --retry` accepts a datetime
  as well as a delay.
- **Refresh maintenance options** `[2026-02]`: rerunning `down` while already
  down refreshes the maintenance options.
- **JSON-aware health and maintenance** `[2026-04]`: the health route can
  return JSON, and `ApplicationBuilder::prefersJsonResponses()` selects
  JSON-preferred behavior.
- **API-aware maintenance** `[2026-07]`: `down` now handles API and JSON routes,
  not only HTML responses.
- **Parallel maintenance state** `[2026-06]`: use the `array` maintenance-mode
  driver to isolate parallel tests.

## Application safety and lifecycle events

- **Bcrypt length enforcement** `[12.0.0]`: configure bcrypt to enforce its
  72-byte input limit instead of silently accepting longer input.
- **Rotated-key MAC validation** `[2026-04]`: decryption authenticates against
  all configured decryption keys, supporting ciphertext created with a rotated
  key.
- **Migration and locale events** `[2025-12]`: `MigrationSkipped` reports
  skipped migrations; `LocaleUpdated` includes the previous locale.
- **Migration names in events** `[2026-05]`: `MigrationStarted` and
  `MigrationEnded` include the migration name.
- **Named event arguments** `[2026-02]`: event classes may be dispatched or
  broadcast with named constructor arguments rather than positional arguments.
- **Request-aware after-response callbacks** `[2026-04]`: callbacks registered
  after the response receive the current request.
