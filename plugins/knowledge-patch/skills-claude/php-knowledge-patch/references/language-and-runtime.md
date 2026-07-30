# Language and Runtime

Use this reference for syntax migrations, object semantics, reflection,
serialization, errors, attributes, constants, closures, and cloning.

## Syntax and scalar operations

### Zero to a negative power (8.4-migration)

`0 ** $negative` and `pow(0, $negative)` are deprecated because they imply
division by zero. Use `fpow()` when IEEE 754 behavior is required.

### Core syntax deprecations (8.5-migration)

Replace `(boolean)`, `(integer)`, `(double)`, and `(binary)` with `(bool)`,
`(int)`, `(float)`, and `(string)`. Terminate `case` labels with `:` rather
than `;`, and call `shell_exec()` instead of using backticks.

### Null offsets and string increments (8.5-migration)

Using `null` as an array offset or as the key passed to `array_key_exists()` is
deprecated; use `''` explicitly when that is the intended key. Incrementing a
non-numeric string with `++` is deprecated; use `str_increment()`.

### Core type and comparison behavior (8.5-migration)

`class_alias()` no longer permits `array` or `callable` as alias names. Loose
comparisons between otherwise uncomparable objects and booleans consistently
follow `(bool)$object`. A final subclass may substitute `static` with `self`
or its concrete class name. `gc_collect_cycles()` no longer counts strings or
resources collected only indirectly through cycles.

### Recursive comparisons now throw (8.4-migration)

Recursion encountered while comparing values throws `Error` instead of ending
with an `E_ERROR` fatal error. Code that intentionally compares cyclic
structures can catch the failure.

## Declarations, constants, and reflection

### Underscore class names (8.4-migration)

Declaring a class whose complete name is `_` is deprecated. Names that merely
start with an underscore remain valid.

### Namespace-block symbol reuse (8.4.0)

Leaving a namespace block clears its seen symbols. A later namespace block may
reuse a symbol name declared in an earlier block without the previous
cross-block conflict.

### Typed extension constants (8.4-migration)

Class constants supplied by Date, Intl, PDO, Reflection, SPL, SQLite, and
XMLReader now declare types. Reflection and tooling must not assume internal
constants are untyped.

### Reflection deprecations (8.5-migration)

The no-op `setAccessible()` methods are deprecated. Do not request a missing
constant with `ReflectionClass::getConstant()` or ask a
`ReflectionProperty` for a default when it has none.

### Attribute target validation (8.5-migration)

Applying `#[\Attribute]` to an abstract class, enum, interface, or trait fails
during compilation. `#[\DelayedTargetValidation]` defers the check until
runtime, where `ReflectionAttribute::newInstance()` can throw.

### Closures and casts in constant expressions (8.5.0)

Closures, first-class callables, and casts may appear in attribute arguments,
property or parameter defaults, constants, and class constants.

```php
const LENGTH = strlen(...);
const ZERO = (int) 0.3;
```

### Attributes on constants (8.5.0)

Attributes may decorate compile-time non-class constants declared with
`const`; `#[\Deprecated]` can mark them.

```php
#[\Deprecated]
const LEGACY_MODE = 1;
```

### Property declaration additions (8.5.0)

`#[\Override]` applies to properties, asymmetric visibility is supported for
static properties, and constructor property promotion may declare final
properties.

## Closures, cloning, and lifecycle hooks

### Closure rebinding restrictions (8.5-migration)

Deprecated closure operations include binding an instance to a static closure,
binding a method to an unrelated object, unbinding `$this`, binding to an
internal-class scope, and rebinding the scope of a closure created from a
function or method.

### Readonly properties during cloning (8.4-migration)

`__clone()` may reinitialize a readonly property but may not take an indirect
reference to it, such as `$ref = &$this->readonlyProperty`.

### Clone-time property replacement (8.5.0)

`clone` has function syntax and accepts a `$withProperties` argument that can
replace properties, including readonly properties, during cloning.

```php
$copy = clone($original, ['id' => $newId]);
```

### Serialization lifecycle hooks (8.4-migration)

`SplFixedArray::__wakeup()` is deprecated. Custom subclasses should implement
`__serialize()` and `__unserialize()`. The nonstandard uppercase `S` tag in
serialized strings is also deprecated.

### Output and magic-method cleanup (8.5-migration)

Producing output inside a user output handler is deprecated; its deprecation
bypasses that handler and remains visible. `__debugInfo()` must return an array
rather than `null`. `__sleep()` and `__wakeup()` are soft-deprecated in favor
of `__serialize()` and `__unserialize()`.

## Errors, shutdown, and cross-extension replacements

### User-generated fatal errors (8.4-migration)

`trigger_error($message, E_USER_ERROR)` is deprecated. Throw an exception when
the failure should be recoverable or call `exit()` for intentional
termination.

### Fatal-error backtraces (8.5.0)

Fatal errors, including maximum-execution-time failures, include a backtrace.

### Core diagnostic cleanup (8.5-migration)

Redeclaring a constant is deprecated in addition to continuing to emit a
warning. The `report_memleaks` INI directive is deprecated.

### Shutdown and class-linking behavior (8.5-migration)

Tick handlers remain active until shutdown functions, destructors, and
output-handler cleanup complete. Traits bind before the parent class.
Compilation and linking errors are delayed until those phases finish; a fatal
error flushes delayed errors without user error handlers. An exception from a
handler processing a linking error no longer prevents linking.

### Core value and filename changes (8.4-migration)

`PHP_DEBUG` and `PHP_ZTS` contain booleans rather than integers. Names generated
for uploads and by `tempnam()` are 13 bytes longer, so revisit path-length
assumptions.

### Legacy overloads replaced by named entry points (8.4-migration)

Replace the ISO-string `DatePeriod` constructor with
`DatePeriod::createFromISO8601String()`. Replace multi-argument Intl calendar
setters and constructors with `setDate()`, `setDateTime()`,
`createFromDate()`, or `createFromDateTime()`. Replace multi-argument
`ldap_connect()` and `ldap_exop()` with `ldap_connect_wallet()` and
`ldap_exop_sync()`, one-argument `ReflectionMethod::__construct()` with
`ReflectionMethod::createFromMethodName()`, and two-argument
`stream_context_set_option()` with `stream_context_set_options()`.

### Deprecated extension entry points (8.4-migration)

Replace `lcg_value()` with `Random\Randomizer::getFloat()`. Pass an array of
function names, such as a flattened `get_defined_functions()` result, to
`SoapServer::addFunction()` instead of `SOAP_FUNCTIONS_ALL` or another integer.
`xml_set_object()` and non-callable method-name strings passed to `xml_set_*()`
are deprecated, as are `CURLOPT_BINARYTRANSFER`, `SUNFUNCS_RET_*`,
`DOM_PHP_ERR`, and obsolete DOM encoding/configuration properties.

### Invalid-input deprecations (8.4-migration)

Passing `null` or `false` to `dba_key_split()` and passing invalid options to
hash functions are deprecated. Validate these inputs instead of relying on
fallback behavior.

### Automatic object cleanup (8.5-migration)

`curl_close()`, `curl_share_close()`, `finfo_close()`, `imagedestroy()`, and
`xml_parser_free()` are deprecated because their handle objects are freed
automatically.

### Extension deprecation sweep (8.5-migration)

Deprecated items include RFC 7231 Date constants, the ignored `context`
argument of `finfo_buffer()`, `MHASH_*`, and `intl.error_level`. Oracle-wallet
LDAP calls and constants, `mysqli_execute()`, and `socket_set_timeout()` are
also deprecated. Use manual Intl error checks or `intl.use_exceptions`,
`mysqli_stmt_execute()`, and `stream_set_timeout()` where applicable.
