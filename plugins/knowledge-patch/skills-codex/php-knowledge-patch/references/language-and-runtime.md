# Language and Runtime

Use this reference for language syntax, execution order, object behavior,
introspection, serialization, diagnostics, and standard-library migration.
Attributions below preserve the included batch identifiers:
`8.4-migration`, `8.4.0`, `8.5-migration`, and `8.5.0`.

## Syntax, names, and constants

### Deprecated spellings and forms

- In `8.5-migration`, `(boolean)`, `(integer)`, `(double)`, and `(binary)` are
  deprecated. Use `(bool)`, `(int)`, `(float)`, and `(string)`.
- A `case` terminated with `;` instead of `:` is deprecated, as are backticks
  used as an alias for `shell_exec()`.
- Declaring a class whose complete name is `_` is deprecated
  (`8.4-migration`); names that merely begin with `_` remain valid.
- Redeclaring a constant is deprecated in addition to emitting a warning, and
  the `report_memleaks` INI directive is deprecated (`8.5-migration`).
- `class_alias()` no longer allows `array` or `callable` as alias names
  (`8.5-migration`).

### Namespace symbol tracking

Since `8.4.0`, leaving a namespace block clears the symbols seen for that
block. A later namespace block can reuse a name declared in an earlier block
without the former cross-block conflict.

### Constant expressions and attributed constants

Since `8.5.0`, closures and first-class callables can appear in attribute
arguments, property and parameter defaults, constants, and class constants.
Casts are also valid constant expressions.

```php
const LENGTH = strlen(...);
const ZERO = (int) 0.3;
```

Attributes can decorate compile-time non-class constants declared with
`const`, and `#[\Deprecated]` can mark them.

```php
#[\Deprecated]
const LEGACY_MODE = 1;
```

## Numeric, key, and comparison behavior

- `0 ** $negative` and `pow(0, $negative)` are deprecated in
  `8.4-migration`, because they imply division by zero. Use `fpow()` only when
  IEEE 754 behavior is intended.
- Using `null` as an array offset or as the key to `array_key_exists()` is
  deprecated in `8.5-migration`; use `''` explicitly when that key is intended.
- Incrementing a non-numeric string with `++` is deprecated; use
  `str_increment()`.
- Loose comparison between otherwise uncomparable objects and booleans now
  consistently follows `(bool) $object` (`8.5-migration`).
- Casting `NAN` to `int`, or casting an unrepresentable float or float-like
  string to `int`, now warns (`8.5-migration`).
- A `printf`-family conversion without explicit precision now treats precision
  as zero instead of resetting it (`8.5-migration`).
- Integer `0` is no longer accepted as `setlocale()`'s locales argument; it
  throws `TypeError` (`8.5-migration`).

## Failure and diagnostic behavior

### Deliberate failures

`trigger_error($message, E_USER_ERROR)` is deprecated in `8.4-migration`.
Throw an exception when recovery should be possible, or call `exit()` when
termination is intentional.

### Recursive comparison and fatal traces

Recursion found while comparing values now throws `Error` instead of ending
with an `E_ERROR` fatal error (`8.4-migration`). Code that deliberately
compares cyclic structures can catch the error.

Since `8.5.0`, fatal errors include a backtrace, including fatal
maximum-execution-time failures. Review logging and redaction paths that
collect fatal output.

### Output handlers and magic methods

In `8.5-migration`:

- Producing output inside a user output handler is deprecated. Its deprecation
  message bypasses that handler so it remains visible.
- `__debugInfo()` must return an array, not `null`.
- `__sleep()` and `__wakeup()` are soft-deprecated in favor of
  `__serialize()` and `__unserialize()`.

`SplFixedArray::__wakeup()` is also deprecated in `8.4-migration`; subclasses
should implement the modern serialization hooks. The nonstandard uppercase
`S` tag in serialized strings is deprecated.

## Closures, cloning, and properties

### Closure binding

The following rebinding operations are deprecated in `8.5-migration`:

- binding an instance to a static closure;
- binding a method to an unrelated object;
- unbinding `$this`;
- binding to an internal-class scope; and
- changing the scope of a closure created from a function or method.

### Readonly properties while cloning

In `8.4-migration`, `__clone()` may reinitialize a readonly property but may
not take an indirect reference to it, such as:

```php
$ref = &$this->readonlyProperty;
```

Since `8.5.0`, `clone` also has function syntax and accepts a
`$withProperties` map. It can replace properties, including readonly
properties, during cloning.

```php
$copy = clone($original, ['id' => $newId]);
```

### Property declaration additions

Since `8.5.0`:

- `#[\Override]` can target properties;
- asymmetric visibility is available for static properties; and
- constructor property promotion can declare final properties.

Final subclasses may also substitute a `static` type with `self` or their
concrete class name (`8.5-migration`).

## Attributes and reflection

Applying `#[\Attribute]` to an abstract class, enum, interface, or trait fails
during compilation in `8.5-migration`. `#[\DelayedTargetValidation]` defers
that check until use, where `ReflectionAttribute::newInstance()` can throw.

The following reflection operations are deprecated in `8.5-migration`:

- no-op `setAccessible()` methods;
- requesting a missing constant with `ReflectionClass::getConstant()`; and
- requesting a default from a `ReflectionProperty` that has no default.

Extension class constants from Date, Intl, PDO, Reflection, SPL, SQLite, and
XMLReader gained declared types in `8.4-migration`. Reflection and source
generation must not assume that internal constants are untyped.

`PHP_DEBUG` and `PHP_ZTS` contain booleans rather than integers in
`8.4-migration`.

## Class linking and shutdown

In `8.5-migration`:

- Traits bind before the parent class.
- Compilation and linking errors are delayed until their respective phases
  finish.
- A fatal error flushes delayed errors without invoking user error handlers.
- An exception thrown while a handler processes a linking error no longer
  prevents linking.
- Tick handlers remain active through shutdown functions, destructors, and
  output-handler cleanup.

Tests that observe autoloading, error order, cleanup, or ticks should exercise
the complete shutdown path.

## SPL and object APIs

### Autoloading and storage

In `8.5-migration`:

- To unregister all autoloaders, iterate over `spl_autoload_functions()`; do
  not pass `spl_autoload_call` to `spl_autoload_unregister()`.
- Prefer `SplObjectStorage::offsetExists()`, `offsetSet()`, and `offsetUnset()`
  over `contains()`, `attach()`, and `detach()`.
- Stop constructing `ArrayObject` or `ArrayIterator` over objects.
- `ArrayObject` no longer accepts enums.

### File objects

`SplFileObject::fwrite()` has a nullable `length` parameter whose default is
`null` rather than `0` in `8.5-migration`.

For `fgetcsv()`, `fputcsv()`, `str_getcsv()`, and the `SplFileObject` CSV
methods, relying on the default `escape` argument is deprecated in
`8.4-migration`. Pass it explicitly unless the file object has already
received an explicit value through `setCsvControl()`.

```php
$row = fgetcsv($stream, null, ',', '"', '\\');
```

Delimiter, enclosure, and escape arguments are validated for valid widths.

## Standard-library inputs and request globals

In `8.5-migration`:

- Pass explicit directory handles to `readdir()`, `rewinddir()`, and
  `closedir()` instead of `null`.
- Restrict `chr()` to integers from 0 through 255 and `ord()` to one-byte
  strings.
- Replace the local `$http_response_header` variable with
  `http_get_last_response_headers()`.
- Deriving `$_SERVER['argc']` and `$_SERVER['argv']` from a query string in a
  non-CLI SAPI is deprecated. Set `register_argc_argv=0`, validate input, and
  read `$_GET` or `$_SERVER['QUERY_STRING']`.
- Destructuring a non-array value other than `null` now warns.

An invalid `round()` mode throws `ValueError` instead of falling back to
`PHP_ROUND_HALF_UP` (`8.4-migration`). `unserialize()` also validates the
`allowed_classes` option.

Since `8.5.0`, `FILTER_THROW_ON_FAILURE` makes filter validation failure throw
instead of returning a failure value. It cannot be combined with
`FILTER_NULL_ON_FAILURE`; the combination throws `ValueError`.

```php
$id = filter_var($input, FILTER_VALIDATE_INT, FILTER_THROW_ON_FAILURE);
```

## Runtime-generated paths and collection counts

Upload names and `tempnam()` names are 13 bytes longer in `8.4-migration`.
Revisit maximum-path and fixed-buffer assumptions.

`gc_collect_cycles()` no longer counts strings or resources that are collected
only indirectly through cycles (`8.5-migration`). Do not treat its count as an
exact inventory of every released value.
