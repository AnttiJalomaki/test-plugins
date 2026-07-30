# Blade, Frontend, Console, and Observability

## Blade, views, and frontend output

- **View timestamp checks** `[2025-04]`: configure view compilation to skip
  cached-view timestamp checks for immutable precompiled deployments.
- **Blade function and constant imports** `[2025-04]`: `@use` accepts PHP
  `function` and `const` modifiers as well as classes.
- **Vite preload limits** `[2025-05]`: cap asset preloads per entry point to
  avoid unbounded preload links.
- **Blade context blocks** `[2025-07]`: `@context('key')` renders when context
  exists and exposes the value as `$value`.
- **Backed-enum dynamic components** `[2025-09]`: pass a `BackedEnum` directly
  to `<x-dynamic-component>` without extracting its scalar.
- **Extendable Vite paths** `[2025-10]`: customize Vite asset-path generation
  through inheritance.
- **Blade stack detection** `[2025-11]`: `@hasStack('scripts')` conditionally
  renders when a stack has pushed content.
- **Isolated includes** `[2026-01]`: `@includeIsolated` does not inherit
  surrounding variables; pass every required value explicitly.
- **Bootstrap 3 pagination names** `[13.0-upgrade]`: direct references must use
  `pagination::bootstrap-3` and `pagination::simple-bootstrap-3` instead of
  the old `default` view names.
- **Font optimization** `[2026-04]`: use the `@fonts` directive with Vite's
  font-optimization runtime.

## Artisan generation and machine-readable output

- **JSON event listings** `[2025-04]`: automate event inspection with
  `php artisan event:list --json`.
- **Combined pruning filters** `[2025-07]`: `model:prune` accepts `--model` and
  `--except` together.
- **Config generation** `[2025-08]`: generate configuration files with
  `php artisan make:config services`.
- **Artisan failure and silence behavior** `[2025-12]`: `cache:clear` returns a
  failure exit code when clearing fails, while `queue:work` respects the
  standard `--quiet` and `--silent` flags.
- **Custom command discovery** `[2025-12]`: customize discovery with
  `commandFileFinder`; tests are excluded, but a real `Command` class whose
  name ends with `Test` remains discoverable.
- **Development command registry** `[2026-06]`: Laravel 13 provides `dev` and
  `dev:list`. Registered commands record source, support priorities, and stop
  peer commands when one fails.

## Command prohibition

- **Queue clear and key generation** `[2026-05]`: `queue:clear` and
  `key:generate` participate in destructive-command prohibition.
- **Cache clear and queue flush** `[2026-06]`: `cache:clear` and `queue:flush`
  also participate.

## Events, discovery, and logging

- **Listener discovery opt-out** `[2026-05]`: auto-discovered event listeners
  can opt out when they should be registered explicitly.
- **Structured JSON logging** `[2026-04]`: Laravel 13's `JsonFormatter` emits
  JSON logs and includes exception context even when the exception handler is
  not bound.
- **Monthly rotation** `[2026-07]`: Laravel 13 provides a monthly log driver and
  matching channel configuration.

## Pagination output

- **Current-page URL** `[2025-05]`: serialized paginator data includes
  `current_page_url`.
- **Link page numbers** `[2025-08]`: serialized paginator link objects include
  a numeric `page` field.
