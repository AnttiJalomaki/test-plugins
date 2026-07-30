# Configuration, Filesystem, and SPL

Use this reference for INI migration, OPcache and JIT, sessions, CSV,
filesystem and stream behavior, and SPL collections and files.

## Runtime and build configuration

### Opcache JIT activation (8.4-migration)

The defaults are `opcache.jit=disable` and
`opcache.jit_buffer_size=64M`. A nonzero buffer alone no longer enables JIT;
set a mode explicitly. When JIT is enabled, compiler initialization failure is
fatal at startup. On 64-bit systems,
`opcache.interned_strings_buffer` has a maximum of `32767`.

```ini
opcache.jit=tracing
opcache.jit_buffer_size=64M
```

### Removed configuration and new core warnings (8.5-migration)

The `disable_classes` INI setting has been removed. Destructuring a non-array
value other than `null` now warns. Casting `NAN`, an unrepresentable float, or
an unrepresentable float-like string to `int` also warns.

### OPcache and native dependency changes (8.5-migration)

OPcache is always built into and loaded with PHP. Its configure flags and
separate module files are gone; legacy `zend_extension=opcache.so` and
`php_opcache.dll` entries warn. `opcache.enable` and `opcache.enable_cli`
remain effective. Intl requires ICU 57.1 or newer. ODBC assumes ODBC 3.5 and
removes driver-specific build flags except DB2, favoring a driver manager on
non-Windows systems.

### Non-CLI argument derivation (8.5-migration)

Deriving `$_SERVER['argc']` and `$_SERVER['argv']` from a query string in
non-CLI SAPIs is deprecated. Set `register_argc_argv=0`; after validating
input, read `$_GET` or `$_SERVER['QUERY_STRING']`.

## Sessions and cookies

### Session configuration cleanup (8.4-migration)

Calling `session_set_save_handler()` with more than two arguments is
deprecated. Stop changing `session.sid_length` and
`session.sid_bits_per_character`, and make storage accept 32-character
hexadecimal IDs. Stop changing deprecated cookie and trans-SID settings:
`session.use_only_cookies`, `session.use_trans_sid`,
`session.trans_sid_tags`, `session.trans_sid_hosts`, and
`session.referer_check`. `SID` is deprecated.

### New warnings for invalid configuration (8.4-migration)

Session configuration warns for a non-positive `session.gc_divisor` or a
negative `session.gc_probability`. `odbc_fetch_row()` also warns and returns
`false` when its row number is zero or negative.

### Session validation (8.5-migration)

Serializing `$_SESSION` with a key containing `|` warns instead of failing
silently. `session_start()` requires its options to be a hashmap and requires
`read_and_close` to have a type compatible with `int`.

### Partitioned cookies (8.5.0)

`session_set_cookie_params()`, `session_get_cookie_params()`,
`session_start()`, `setcookie()`, and `setrawcookie()` recognize the
`partitioned` option.

```php
setcookie('sid', $value, ['partitioned' => true]);
```

## CSV and standard input handling

### CSV escape arguments (8.4-migration)

Relying on the default `escape` argument is deprecated for `fgetcsv()`,
`fputcsv()`, `str_getcsv()`, and their `SplFileObject` equivalents. Pass it
explicitly unless an `SplFileObject` already received an explicit value
through `setCsvControl()`.

```php
$row = fgetcsv($stream, null, ',', '"', '\\');
```

### Standard and XML exception tightening (8.4-migration)

An invalid `round()` mode throws `ValueError` instead of behaving as
`PHP_ROUND_HALF_UP`. CSV delimiter, enclosure, and escape widths;
`php_uname()` modes; and `unserialize()`'s `allowed_classes` option are
validated. XMLReader, XMLWriter, and XSL operations throw for invalid
encodings, null bytes, incompatible objects, or failed PHP callbacks where
applicable.

### Standard-library input deprecations (8.5-migration)

Pass an explicit directory handle to `readdir()`, `rewinddir()`, and
`closedir()` rather than `null`. Restrict `chr()` to integers from 0 through
255 and `ord()` to one-byte strings. Replace local
`$http_response_header` use with `http_get_last_response_headers()`.

## Streams and filesystem objects

### Stream buckets and constructor failures (8.4-migration)

`stream_bucket_make_writeable()` and `stream_bucket_new()` return
`StreamBucket` instead of `stdClass`. A failed `Tidy` construction throws
instead of leaving a broken object after a warning.

### Configurable Readline history path (8.4.0)

`PHP_HISTFILE` changes the path used for `.php_history`.

```sh
PHP_HISTFILE=/path/to/.php_history
```

### Locking zlib streams (8.5.0)

`flock()` supports zlib streams instead of always failing to lock them.

## SPL collections and files

### SPL legacy APIs (8.5-migration)

To unregister every autoloader, iterate over `spl_autoload_functions()` instead
of passing `spl_autoload_call` to `spl_autoload_unregister()`. Prefer
`SplObjectStorage::offsetExists()`, `offsetSet()`, and `offsetUnset()` over
`contains()`, `attach()`, and `detach()`. Stop constructing `ArrayObject` or
`ArrayIterator` over objects.

### SPL object and file behavior (8.5-migration)

`ArrayObject` no longer accepts enums. `SplFileObject::fwrite()` has a nullable
`length` parameter whose default is `null` rather than `0`.

