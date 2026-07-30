# Collections, Strings, Support APIs, and JSON Schema

## Collections and lazy collections

- **Multiline `Str::is()`** `[12.0.0]`: pattern matching spans line breaks in a
  multiline string.
- **Stepped lazy ranges** `[12.0.0]`: `LazyCollection::range($from, $to, $step)`
  accepts a step.
- **Reindexed chunks** `[2025-03]`: pass `preserveKeys: false` to
  `Collection::chunk()` to reindex each chunk.
- **Callback single-item checks** `[2025-05]`:
  `containsOneItem($callback)` reports whether exactly one item matches.
- **Static higher-order calls** `[2025-06]`: higher-order proxies can call
  static methods when items are class strings.
- **Closure plucks** `[2025-07]`: both the value and key arguments of
  `Collection::pluck()` may be closures.
- **Lazy heartbeats** `[2025-08]`: `LazyCollection::withHeartbeat()` invokes a
  liveness or lease-renewal callback during consumption.
- **Enum keys** `[2025-08]`: `Collection::keyBy()` normalizes a returned enum to
  an array-compatible key.
- **Enum grouping** `[2025-09]`: `countBy()` callbacks may return enums, and
  `groupBy()` accepts `UnitEnum` values.
- **Lazy timeout callbacks** `[2025-09]`:
  `LazyCollection::takeUntilTimeout()` accepts follow-up work to run at timeout.
- **Enum sorting** `[2026-04]`: collection and `Arr` sorting accept
  `SortDirection`.
- **Reduction into an accumulator** `[2026-07]`: use `reduceInto()` when the
  reduction builds into an accumulator object or value.

## Arrays, strings, numbers, and Fluent

- **Typed array getters** `[2025-04]`: `Arr` has string, integer, float,
  boolean, and array getters that enforce the requested type.
- **Unicode-aware trimming** `[2025-05]`: `Str::trim()` removes the complete set
  of invisible characters, including formatting and zero-width characters.
- **Locale number parsing** `[2025-05]`: `Number::parseInt()` and
  `parseFloat()` accept locale-formatted grouping and decimal separators.
- **Number parse failures** `[2025-09]`: both number parsers can return `false`;
  check failure before using the result as numeric.
- **Iterable Fluent** `[2025-07]`: `Fluent` implements iteration.
- **Depth-limited flattening** `[2026-03-laravel-12]`: `Arr::dot($array,
  depth: 2)` limits recursive flattening.
- **Carbon retry sleeps** `[2026-03-laravel-12]`: `retry()` accepts
  `CarbonInterval` sleep durations.
- **Carbon overflow control** `[2026-04]`: `plus()` and `minus()` accept an
  `overflow` option for date rollover behavior.
- **Safe cursor decoding** `[2026-04]`: `Cursor::fromEncoded()` returns `null`
  for malformed payloads.
- **Case normalization** `[2026-05]`: `Str::studly()` and `Str::pascal()` accept
  `normalize:` before conversion.
- **Count-aware strings** `[2026-07]`: `Str` and `Stringable` provide
  `counted()`.
- **Iterable array helpers** `[2026-07]`: `Arr::every()`, `some()`, and
  `last()` accept iterables.

## URIs, JSON, and translation

- **Macroable URIs** `[2025-04]`: extend `Illuminate\Support\Uri` through
  `Uri::macro()`.
- **Custom encoder flags** `[2025-04]`: callbacks assigned to `Json::$encoder`
  receive the active JSON encoding flags.
- **JSON-serializable URIs** `[2025-07]`: `Uri` implements `JsonSerializable`.
- **Literal translation delimiters** `[2026-01]`: translation lines may contain
  square or curly braces without being rejected merely as apparent choice
  delimiters.
- **Typed translation access** `[2026-06]`: use typed translation accessors
  instead of manually narrowing a general return value.

## JSON Schema

- **Schema contract** `[2025-11]`: depend on the JSON Schema contract when
  extending schema generation rather than coupling to an implementation.
- **Member dependencies** `[2025-12]`: schemas can express dependencies between
  members.
- **Numeric and array constraints** `[2026-02]`: numeric schema types support
  `multipleOf`; array types support `uniqueItems`.
- **Unset fluent flags** `[2026-05]`: fluent boolean flags can be cleared after
  being enabled.
- **Deserialization and composition** `[2026-06]`: deserialize array schemas
  and multi-type unions, and compose schemas with `anyOf`.
