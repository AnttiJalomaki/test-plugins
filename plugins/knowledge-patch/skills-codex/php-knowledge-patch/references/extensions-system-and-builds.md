# Extensions, System APIs, and Builds

Use this reference for migrated entry points, handle cleanup, validation and
failure contracts, process and socket APIs, OPcache, and native build
requirements. Attributions retain `8.4-migration`, `8.4.0`,
`8.5-migration`, and `8.5.0`.

## Named replacements for legacy overloads

Replace ambiguous overloads deprecated in `8.4-migration`:

| Deprecated form | Replacement |
| --- | --- |
| ISO-string `DatePeriod` constructor | `DatePeriod::createFromISO8601String()` |
| Multi-argument Intl calendar setters | `setDate()` or `setDateTime()` |
| Multi-argument Intl calendar constructors | `createFromDate()` or `createFromDateTime()` |
| Multi-argument `ldap_connect()` | `ldap_connect_wallet()` |
| Multi-argument `ldap_exop()` | `ldap_exop_sync()` |
| One-argument `ReflectionMethod::__construct()` | `ReflectionMethod::createFromMethodName()` |
| Two-argument `stream_context_set_option()` | `stream_context_set_options()` |

## Deprecated extension entry points

In `8.4-migration`:

- Replace `lcg_value()` with `Random\Randomizer::getFloat()`.
- Validate inputs before `dba_key_split()` rather than passing `null` or
  `false`.
- Validate hash options instead of relying on old fallback behavior.
- `CURLOPT_BINARYTRANSFER`, the `SUNFUNCS_RET_*` constants, `DOM_PHP_ERR`,
  and obsolete DOM encoding and configuration properties are deprecated.

The XML and SOAP migrations are detailed in
[XML, SOAP, text, and media](xml-soap-text-and-media.md).

## Automatic object cleanup

These close or destroy calls are deprecated in `8.5-migration`, because their
handle objects are freed automatically:

- `curl_close()`;
- `curl_share_close()`;
- `finfo_close()`;
- `imagedestroy()`; and
- `xml_parser_free()`.

Remove calls made only for lifetime management. Keep application-level cleanup
that has separate semantics.

## General validation shifts

In `8.4-migration`, invalid numeric ranges, domains, encodings, modes, and
options increasingly throw `ValueError`. Affected examples include:

- `curl_multi_select()` timeouts;
- GD quality, speed, scale, and filter arguments;
- empty gettext domains;
- Intl locales and `ResourceBundle` offsets;
- malformed mbstring maps or encodings;
- invalid `round()` modes;
- CSV delimiter, enclosure, and escape widths;
- `php_uname()` modes; and
- `unserialize()` `allowed_classes`.

Callers should validate documented domains or catch the specific exception
instead of depending on warnings or coercion.

In `8.5-migration`, stricter validation also covers:

- `bzcompress()` block size 1–9 and work factor 0–250;
- Intl timezone and locale operations;
- LDAP options;
- `pcntl_exec()` arguments and environment;
- POSIX limits;
- SNMP host, port, timeout, and retry values;
- socket ports, hints, and multicast contexts; and
- Tidy configuration key types, invalid settings, and read-only settings.

## PCNTL signals and runtime failures

In `8.4-migration`, signal-mask and signal-wait APIs reject:

- empty or non-integer signal lists;
- invalid signal numbers;
- invalid mask modes; and
- invalid timed-wait durations.

They throw `TypeError` or `ValueError` for invalid input. Runtime failures
consistently return `false`, never `-1`.

## Tidy construction

A failed `Tidy` construction throws in `8.4-migration` instead of emitting a
warning and leaving a broken object.

## LDAP and socket migrations

Oracle-wallet LDAP calls and their constants are deprecated in
`8.5-migration`. The named LDAP wallet and synchronous extended-operation
entry points replace the older multi-argument overloads where applicable.

`socket_set_timeout()` is deprecated; use `stream_set_timeout()`.

## Other extension deprecations

In `8.5-migration`:

- RFC 7231 Date constants are deprecated.
- The ignored `context` argument of `finfo_buffer()` is deprecated.
- `MHASH_*` constants are deprecated.
- `intl.error_level` is deprecated; check Intl errors explicitly or use
  `intl.use_exceptions`.
- `mysqli_execute()` is deprecated; use `mysqli_stmt_execute()`.
- The `disable_classes` INI setting has been removed.

## GMP restrictions

`GMP` is final and cannot be subclassed in `8.4-migration`.

## OPcache and JIT

### JIT activation

In `8.4-migration`, the defaults are:

```ini
opcache.jit=disable
opcache.jit_buffer_size=64M
```

A nonzero buffer alone no longer enables JIT; set an explicit mode such as
`opcache.jit=tracing`. When JIT is enabled, compiler initialization failure is
startup-fatal. On 64-bit systems, the maximum
`opcache.interned_strings_buffer` value is `32767`.

### Integrated OPcache

In `8.5-migration`, OPcache is always built into and loaded with PHP. Configure
flags and separate module files are gone. Legacy
`zend_extension=opcache.so` or `php_opcache.dll` entries warn, although
`opcache.enable` and `opcache.enable_cli` remain effective.

## Native dependency changes

In `8.5-migration`:

- Intl requires ICU 57.1 or newer.
- ODBC assumes ODBC 3.5.
- ODBC driver-specific build flags are removed except for DB2; use a driver
  manager on non-Windows systems.

In `8.4-migration`, building PDO_FIREBIRD requires a C++ compiler and fbclient
3.0 or newer.

SOAP's optional session dependency and `--enable-rtld-now` startup interaction
are covered in [XML, SOAP, text, and media](xml-soap-text-and-media.md).

## Internal data and constants

Class constants supplied by Date, Intl, PDO, Reflection, SPL, SQLite, and
XMLReader have declared types in `8.4-migration`. Tools using reflection must
preserve that metadata.

`PHP_DEBUG` and `PHP_ZTS` are booleans rather than integers.

Successful DBA and ODBC handles are objects, as detailed in
[Database, sessions, and persistence](database-sessions-and-persistence.md).
