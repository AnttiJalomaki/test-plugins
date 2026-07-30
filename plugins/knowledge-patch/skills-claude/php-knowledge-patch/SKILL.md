---
name: php-knowledge-patch
description: PHP
version: 8.5.0
license: MIT
metadata:
  author: Nevaberry
---


# PHP Knowledge Patch

Use this skill when upgrading PHP applications, extensions, build images, or
runtime configuration, and when adopting recent language or extension APIs.
Start with the migration checks below, then open the reference that matches the
code being changed.

## Reference index

| Reference | Topics |
| --- | --- |
| [Language and runtime](references/language-and-runtime.md) | Syntax, types, object behavior, reflection, serialization, errors, attributes, closures, constants, cloning |
| [Configuration, filesystem, and SPL](references/configuration-filesystem-and-spl.md) | INI changes, OPcache, sessions, CSV, streams, filesystem APIs, SPL collections and files |
| [Databases and PDO](references/databases-and-pdo.md) | PDO and driver subclasses, MySQLi, PostgreSQL, SQLite, Firebird, DBA, ODBC |
| [Networking, crypto, and processes](references/networking-crypto-and-processes.md) | cURL, OpenSSL, LDAP, PCNTL, sockets, filters, mail, native dependencies |
| [XML, SOAP, and XSL](references/xml-soap-and-xsl.md) | DOM, XML handlers, SimpleXML, XMLReader/Writer, SOAP, XPath, XSLT |
| [Text, internationalization, and media](references/text-intl-and-media.md) | PCRE, mbstring, Intl, locale, formatting, GD, EXIF, HEIF, SVG, Tidy |

## Upgrade triage

1. Run the test suite with all deprecations visible:

   ```ini
   error_reporting=E_ALL
   display_errors=1
   ```

2. Search for removed or deprecated syntax and APIs:

   ```sh
   rg '\((boolean|integer|double|binary)\)|`[^`]*`'
   rg 'trigger_error\s*\([^,]+,\s*E_USER_ERROR'
   rg '\b(__sleep|__wakeup|setAccessible)\s*\('
   rg '\b(curl_close|curl_share_close|finfo_close|imagedestroy|xml_parser_free)\s*\('
   ```

3. Audit calls whose invalid-input behavior changed from warnings, coercion, or
   sentinel values to `TypeError`, `ValueError`, or another exception.

4. Inspect every PHP and extension INI file. Remove obsolete extension-loading
   directives and deprecated session settings before evaluating runtime
   behavior.

5. Test database connections with the production driver and server versions;
   driver-specific parsing, credential precedence, fetch flags, and timeout
   codes can change behavior without a syntax error.

6. Exercise shutdown paths, output handlers, serialization, cloning, XML
   callbacks, and cyclic comparisons explicitly. These paths often escape
   ordinary request tests.

## Highest-priority deprecations

### Replace legacy language forms

- Use `(bool)`, `(int)`, `(float)`, and `(string)` instead of the long cast
  aliases.
- End `case` labels with `:`, not `;`.
- Call `shell_exec()` explicitly instead of using backticks.
- Replace non-numeric string `++` with `str_increment()`.
- Use the empty string explicitly instead of `null` when an empty array key is
  intended.
- Replace `trigger_error(..., E_USER_ERROR)` with an exception for recoverable
  failure or `exit()` for intentional termination.

### Modernize lifecycle hooks

- Prefer `__serialize()` and `__unserialize()` to `__sleep()` and
  `__wakeup()`.
- Make `__debugInfo()` return an array.
- Do not produce output recursively from inside an output handler.
- Let handle objects clean themselves up; explicit close/free calls for cURL,
  file-info, GD image, and XML parser objects are deprecated.

### Make implicit arguments explicit

- Supply the CSV `escape` argument to `fgetcsv()`, `fputcsv()`, `str_getcsv()`,
  and matching `SplFileObject` methods.
- Pass a real directory handle to `readdir()`, `rewinddir()`, and
  `closedir()`.
- Use three-argument PostgreSQL fetch/field calls with `row: null`.
- Pass no `key_length` to `openssl_pkey_derive()`.
- Stop using a query string to synthesize non-CLI `argc` and `argv`.

### Move to named or driver-specific APIs

- Replace ambiguous legacy overloads with their named DatePeriod, Intl, LDAP,
  Reflection, and stream-context entry points.
- Prefer `Pdo\*` driver constants and driver-subclass methods to prefixed
  members on base `PDO`.
- Replace deprecated MySQLi administrative helpers with SQL where appropriate,
  and use `mysqli_stmt_execute()` instead of `mysqli_execute()`.
- Replace SPL collection aliases such as `contains()`, `attach()`, and
  `detach()` with their `offset*()` forms.

## Behavior changes to test

### Exceptions and warnings

- Recursive comparison now throws `Error`.
- Many extension APIs reject invalid ranges, modes, encodings, ports, locales,
  signal lists, configuration keys, or callback results with exceptions.
- Failed `Tidy` construction throws.
- Invalid SimpleXML XPath result kinds warn and return `false`.
- Destructuring non-arrays and unrepresentable float-to-integer casts warn.
- Sendmail transport failures now warn and make `mail()` return `false`.

Catch only errors the application can handle. Validate user input before
calling an extension rather than depending on former fallback behavior.

### Objects replace resources

DBA, ODBC, SOAP internals, and stream buckets use dedicated objects. Replace
`is_resource()` checks with the documented object type or an explicit
`=== false`/null failure check. Do not assume a handle-closing function is still
the lifetime boundary.

### Iteration and fetch state

- SimpleXML helper calls and string casts no longer rewind iteration.
- PDO rejects fetch-mode changes during a fetch and enforces valid fetch-flag
  combinations.
- `PDO::FETCH_CLASS` constructor arguments follow ordinary named-argument and
  by-reference rules.

### Runtime and build configuration

- A nonzero JIT buffer alone does not activate JIT; set a JIT mode.
- OPcache is built in, so remove legacy module-loading lines while retaining
  `opcache.enable` settings.
- Remove `disable_classes`; it no longer exists.
- Review updated ICU, ODBC, Firebird, SOAP/session, and OpenSSL requirements
  before rebuilding PHP.

## High-value additions

### Constant expressions and properties

Closures, first-class callables, and casts can be used in constant expressions
and defaults. Non-class constants can carry attributes. Properties gain more
precise declaration tools: property-level `#[\Override]`, asymmetric
visibility for static properties, and final promoted properties.

Clone-time property replacement can update selected properties, including
readonly properties:

```php
$copy = clone($original, ['id' => $newId]);
```

Use this in value-object update methods, but keep constructor and clone
invariants aligned.

### Failure-oriented APIs

`FILTER_THROW_ON_FAILURE` converts filter validation failures to exceptions:

```php
$id = filter_var($input, FILTER_VALIDATE_INT, FILTER_THROW_ON_FAILURE);
```

Do not combine it with `FILTER_NULL_ON_FAILURE`.

Fatal errors now include backtraces, improving crash diagnosis. Keep log
redaction and size limits in mind because traces can expose arguments and
application paths.

### cURL capabilities and redirects

Use `curl_version()['feature_list']` for named capability checks. Pre-request
and debug callbacks add connection-time policy and observability. Select an
explicit redirect mode when `true` is too broad, use persistent share handles
for safe cross-request connection reuse, and select the large file-size option
on platforms where the older option is 32-bit.

### Database and session capabilities

- `PDO::connect()` and direct construction can return driver-specific
  subclasses.
- SQLite can set deferred, immediate, or exclusive transaction behavior for
  later `beginTransaction()` calls.
- Cookie APIs accept the `partitioned` option.

Verify browser requirements when setting partitioned cookies; the option does
not replace other required cookie attributes.

### Internationalization and document processing

- `IntlListFormatter` formats localized conjunction, disjunction, and unit
  lists when the required ICU is present.
- DOM XPath and XSLT registration APIs accept native callables.
- Namespaced SOAP class maps, namespace-aware XSL parameters, SOAP reason
  languages, and selectable URI parsers reduce ad hoc protocol handling.
- Image inspection recognizes more metadata, HEIF/HEIC, and optionally SVG;
  consume the reported dimension units rather than assuming pixels.

## Review checklist

- [ ] Deprecations are visible in CI and no handler hides them.
- [ ] Deprecated syntax, overloads, constants, methods, and INI settings are
      removed or deliberately isolated.
- [ ] Invalid-input paths are tested for their new exception or warning type.
- [ ] Resource checks account for migrated object handles.
- [ ] JIT and OPcache configuration is explicit and free of obsolete loading
      directives.
- [ ] PDO behavior is tested per driver, including parser, DSN, fetch, and
      transaction differences.
- [ ] Session identifiers, serialization keys, and cookie options work with
      the configured storage and clients.
- [ ] XML, SOAP, XSL, PCRE, mbstring, and Intl behavior is covered with real
      production data.
- [ ] Build images satisfy current extension dependency floors.
- [ ] New callbacks and fatal backtraces do not leak secrets into logs.

