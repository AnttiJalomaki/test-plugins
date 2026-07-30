# Validation and Testing

## File, identifier, and array validation

- **SVG image validation** `[12.0-upgrade]`: `image` rejects SVG by default.
  Opt in with `image:allow_svg` or `File::image(allowSvg: true)` after assessing
  SVG security.
- **Versioned UUID validation** `[12.0.0]`: constrain the `uuid` rule with a
  version, including version 2 through the maximum defined UUID version:
  `'id' => ['required', 'uuid:7']`.
- **Alternative rule sets** `[2025-04]`: `Rule::anyOf()` accepts multiple
  complete rule sets and passes when any set passes.
- **Required array keys** `[2025-05]`: `in_array_keys:email,phone` requires at
  least one listed key to exist in the input array.
- **Required array values** `[2025-05]`: `Rule::contains([...])` requires an
  array to contain the supplied values.
- **Strict booleans and numbers** `[2025-07]`: `boolean:strict` accepts actual
  booleans only; `numeric:strict` accepts integers and floats, not numeric
  strings.
- **Ordinal wildcard messages** `[2025-09]`: use `:ordinal-position` in
  wildcard array validation messages; fallback remains safe without Intl.
- **Capitalized placeholders** `[2025-10]`: replacement recognizes capitalized
  placeholder forms in custom messages.
- **Uploaded-file encoding** `[2025-11]`: validate text upload encoding with a
  rule such as `encoding:UTF-8`.
- **Precognitive wildcard rules** `[2026-01]`: precognitive requests support
  wildcard paths in array validation.
- **Strict fluent integers** `[2026-03-laravel-12]`: use
  `Rule::numeric()->integer(strict: true)` to reject loosely integer-like
  values.
- **Fluent string and conditional rules** `[2026-03-laravel-12]`: use the
  fluent string builder and completed conditional rule builders, for example
  `Rule::string()->min(3)->max(100)`.
- **Fake DNS** `[2026-07]`: fake validation-rule DNS lookups in tests.

## Validation lifecycle and form requests

- **Fluent password semantics** `[2025-12]`: `Password::required()` and
  `sometimes()` correctly distinguish missing, nullable empty, and rule-array
  cases.
- **Validator outcome callbacks** `[2026-02]`: register result-dependent work
  with `whenFails()` and `whenPasses()`.
- **Strict form requests** `[2026-04]`: strict mode includes
  `failOnUnknownFields` for query parameters; call
  `FormRequest::flushState()` to reset global strict state between tests.
- **Stringified password rules** `[2026-05]`: a fluent `Password` rule can
  convert itself to its equivalent password-rule string.

## Framework and bootstrap testing

- **PHPUnit 12.2** `[2025-06]`: Laravel 12 supports PHPUnit 12.2 in addition to
  the earlier PHPUnit 11 baseline.
- **PHPUnit 12.4** `[2025-10]`: Laravel 12 supports PHPUnit 12.4.
- **Cached bootstrap traits** `[2025-11]`: use `WithCachedRoutes` and
  `WithCachedConfig` to exercise route and configuration cache bootstrapping;
  cached-config tests support parallel runs.
- **Parallel pre-migration hook** `[2025-12]`: prepare a selected parallel-test
  database after selection but before migrations run.
- **Parallel cache isolation** `[2026-02]`: parallel tests receive
  process-specific cache prefixes by default; disable this only when testing a
  deliberately shared cache.
- **Binary response assertions** `[2026-03-laravel-12]`: normal HTTP response
  assertions support `BinaryFileResponse`.
- **Test-scoped string factories** `[13.0-upgrade]`: UUID, ULID, and random
  string factories registered through `Str` reset at teardown; register them
  in each relevant test or setup hook.
- **Test-time isolation** `[2026-07]`: fake time is reset globally after every
  test.

## Fakes, assertions, and request inspection

- **Scoped queue fakes** `[2025-07]`: `Queue::fakeFor()` limits faking to a
  callback; `fakeExceptFor()` lets selected jobs use the real queue.
- **Waiting on fake processes** `[2025-10]`: `FakeInvokedProcess::waitUntil()`
  tests condition-based process waiting with the production API.
- **Enum-aware integration points** `[2025-12]`: translator replacement values
  may be enums, and `Storage::fake()` accepts an enum disk name.
- **Exact queue assertions** `[2026-01]`: the public
  `QueueFake::assertPushedTimes()` asserts an exact dispatch count.
- **Queue fake inspection** `[2026-07]`: inspect delayed and reserved jobs,
  register `beforePushing()` and `afterPushing()` hooks, and use
  `creationTimeOfOldestPendingJob()` without a backend.
- **Iterable database-empty assertions** `[2026-07]`:
  `assertDatabaseEmpty()` accepts multiple database targets as an iterable.
- **HTTP fake stream bodies** `[2026-07]`: fake HTTP responses accept stream
  bodies for streamed-content tests.
