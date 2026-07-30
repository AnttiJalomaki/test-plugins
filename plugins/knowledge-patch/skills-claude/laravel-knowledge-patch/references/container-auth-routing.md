# Container, Authentication, Routing, and Context

## Contents

- [Container resolution and bindings](#container-resolution-and-bindings)
- [Context, exceptions, and application state](#context-exceptions-and-application-state)
- [Authentication and request input](#authentication-and-request-input)
- [Routing, middleware, and URLs](#routing-middleware-and-urls)
- [Rate limiting and conditional helpers](#rate-limiting-and-conditional-helpers)

## Container resolution and bindings

- **Optional constructor defaults** `[12.0-upgrade]`: the container leaves a
  resolvable optional class parameter at its declared default. For
  `public ?Carbon $date = null`, `resolve()` supplies `null`.
- **Return-type inferred bindings** `[12.0.0]`: omit the abstract when a binding
  closure declares it as the return type:

  ```php
  $app->bind(fn (): ServiceContract => new Service);
  ```

- **Contextual context injection** `[2025-05]`: `#[Context('trace_id',
  hidden: true)]` injects a value from hidden application context.
- **Attribute interface bindings** `[2025-06]`: declare an interface binding
  with `#[Bind(RedisEventPusher::class)]`.
- **Singleton and scoped attributes** `[2025-07]`: `#[Singleton]` and
  `#[Scoped]` declare class lifetime without a provider binding.
- **Enum selectors in attributes** `[2025-08]`: `Bind` accepts a `UnitEnum`
  environment selector and the `Database` contextual attribute accepts an enum
  connection selector.
- **String bindings in `Give`** `[2025-11]`: `#[Give('cache.store')]` can inject
  a string-named service, not only a class.
- **Nullable defaults in method calls** `[13.0-upgrade]`: `Container::call()`
  honors the default of an unbound nullable class parameter, matching
  constructor injection.
- **Enum selectors across managers** `[2026-04]`: queue, logging, cache, mail,
  auth, password, broadcast, notification, and concurrency manager methods
  accept enums. Enum selectors also cover several default-driver setters,
  Redis purging, limiter names, and cache-touch keys.
- **Parameter-aware contextual attributes** `[2026-06]`: a contextual
  attribute's resolver receives the target `ReflectionParameter`.
- **Conditional bindings** `[2026-07]`: use `BindWhen` for conditionally
  binding services.

## Context, exceptions, and application state

- **Scoped context** `[2025-03]`: `Context::scope($callback)` restores the
  surrounding context after the callback:

  ```php
  Context::scope(function () {
      Context::add('tenant_id', 123);
  });
  ```

- **Selective log-context removal** `[2025-03]`: call
  `Log::withoutContext(['tenant_id', 'trace_id'])` to remove selected context
  values rather than clearing everything.
- **Remembered context** `[2025-07]`: `Context::remember()` and
  `rememberHidden()` lazily compute and save a value only when the key is
  missing.
- **Report suppression callbacks** `[2025-07]`: use `dontReportUsing()` when
  exception reporting suppression needs a callback instead of a class list.
- **Scheduled context propagation** `[2025-11]`: scheduled tasks inherit
  Laravel context from the scheduling process.

## Authentication and request input

- **Nested request merging** `[12.0-upgrade]`:
  `Request::mergeIfMissing()` treats dot notation as a nested array path;
  `'user.last_name'` no longer creates a literal dotted top-level key.
- **Nested policy discovery** `[12.0.0]`: policy discovery follows parallel
  nested namespaces, such as `App\Models\Admin\User` to
  `App\Policies\Admin\UserPolicy`.
- **Enum input defaults** `[2025-05]`: `Request::enum()` accepts a default enum
  returned when the key is absent or invalid.
- **Session regeneration on login** `[2025-10]`: `Auth::login()` regenerates
  the session identifier; update manual-login tests accordingly.
- **Fluent request defaults** `[2025-11]`: `Request::fluent($key, $default)`
  accepts a default for missing input.
- **Remember-cookie payloads** `[2026-01]`: remember cookies store a MAC of the
  password hash, not the raw hash. Do not parse or create the old payload.
- **Expanded enum integration** `[2026-01]`: session and cache APIs accept enum
  keys more broadly, including session `now()` and `flash()` and cache
  `flexible()` and `withoutOverlapping()`. Authorization abilities accept
  `UnitEnum`; `PendingBatch::onConnection()`, `Storage::persistentFake()`, and
  related selectors also accept enums.
- **Deprecated request getter** `[2026-02]`: replace `Request::get()` with an
  explicit source accessor such as `input()` or `query()`.
- **Wildcard trim exclusions** `[2025-12]`: `TrimStrings` exclusions accept
  wildcard patterns, including nested input paths.

## Routing, middleware, and URLs

- **Duplicate route-name precedence** `[12.0-upgrade]`: cached and uncached
  routes now both select the first registered route when names are duplicated.
- **Signed URL exclusions** `[12.0.0]`: signed URL validation accepts a callback
  that selects query-string parameters to ignore.
- **New macro extension points** `[2025-09]`: `RouteRegistrar` and
  `Illuminate\Support\Benchmark` support the normal `macro()` API.
- **Middleware-filtered route lists** `[2025-11]`: narrow output with
  `php artisan route:list --middleware=auth`.
- **Closure locations in route lists** `[2026-03-laravel-12]`: `route:list`
  displays the file and line for closure routes.
- **Domain route precedence** `[13.0-upgrade]`: explicit-domain routes match
  before routes without domains regardless of registration order.
- **Request forgery protection** `[13.0-upgrade]`: use
  `PreventRequestForgery`, which checks CSRF tokens and `Sec-Fetch-Site`.
  `VerifyCsrfToken` and `ValidateCsrfToken` remain deprecated aliases; configure
  with `preventRequestForgery(...)`.
- **Expanded PHP attributes** `[13.0.0]`: use controller `Middleware` and
  `Authorize` attributes for colocated request policy; queue jobs support
  `Tries`, `Backoff`, `Timeout`, and `FailOnTimeout`.
- **Inherited framework attributes** `[2026-04]`: controller `Middleware`,
  queued `WithoutRelations`, model `CollectedBy`, and declarations on abstract
  parents are inherited. A child model's `Table` attribute overrides its
  parent's.
- **CORS bypass callbacks** `[2026-01]`: exempt selected requests with
  `HandleCors::skipWhen($callback)`.
- **Restricted route unserialization** `[2026-06]`: serialized routes only
  instantiate permitted classes; do not store arbitrary restorable objects in
  route values.
- **Parameter-name route injection** `[2026-06]`: `RouteParameter` can infer the
  route key from the attributed parameter name.
- **Route metadata** `[2026-06]`: attach application or tooling annotations
  directly to routes.
- **Controller exclusions** `[2026-07]`: use `WithoutMiddleware` on controllers
  to declare middleware exclusions.

## Rate limiting and conditional helpers

- **Response-aware rate limits** `[2025-09]`: use `Limit::after()` to decide
  from the response whether a completed request counts against the limit.
- **Macroable rate limiter** `[2026-07]`: `RateLimiter` supports application
  macros.
- **Closure conditions for `when()`** `[12.0.0]`: a closure condition is
  evaluated before choosing the applicable branch.
- **Lazy `throw_if()` exceptions** `[2025-10]`: pass a closure to construct an
  exception only when the condition is true.
