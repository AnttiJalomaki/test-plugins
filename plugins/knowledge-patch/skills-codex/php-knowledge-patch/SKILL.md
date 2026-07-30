---
name: php-knowledge-patch
description: PHP
version: 8.5.0
license: MIT
metadata:
  author: Nevaberry
---


# PHP Knowledge Patch

Use this skill when upgrading, writing, reviewing, or debugging PHP code,
extensions, runtime configuration, or deployment builds. Inspect the actual PHP
version, loaded extensions, linked native libraries, SAPI, and database drivers
before applying version-dependent guidance.

Start with the migration checks below, then open the topic reference that
matches the code or configuration being changed. Prefer installed behavior,
project tests, and runtime reflection over assumptions about internal types,
constants, defaults, or error modes.

## Reference index

| Reference | Topics |
| --- | --- |
| [Language and runtime](references/language-and-runtime.md) | Syntax, constants, closures, properties, attributes, object behavior, serialization, reflection, SPL, diagnostics, and standard-library changes |
| [Database, sessions, and persistence](references/database-sessions-and-persistence.md) | PDO, MySQLi, PostgreSQL, Firebird, SQLite, DBA, ODBC, sessions, cookies, and database migration traps |
| [Networking, crypto, patterns, and I/O](references/networking-crypto-patterns-and-io.md) | cURL, OpenSSL, PCRE, streams, Readline, mail, files, and runtime capability checks |
| [XML, SOAP, text, and media](references/xml-soap-text-and-media.md) | DOM, XMLReader, XMLWriter, XSL, SimpleXML, SOAP, mbstring, Intl, image metadata, and callbacks |
| [Extensions, system APIs, and builds](references/extensions-system-and-builds.md) | Renamed entry points, removed handles, validation, PCNTL, LDAP, sockets, OPcache, native dependencies, and extension constants |

## Working method

1. Read `composer.json`, platform constraints, container images, CI matrices,
   and production runtime metadata to identify the effective PHP version.
2. Enumerate loaded extensions and their linked libraries; features such as
   Argon2, list formatting, SVG sizing, and newer key types depend on build
   capabilities.
3. Search for deprecated syntax and APIs before changing error handling.
   Several formerly permissive paths now warn, throw `TypeError` or
   `ValueError`, or return `false`.
4. Audit checks based on `is_resource()`, integer-valued constants, integer
   PDO attributes, implicit iterator rewinds, and destructor-like close calls.
5. Run tests with deprecations visible, exercise invalid-input paths, and test
   startup configuration separately from request behavior.
6. Verify behavior against every supported runtime rather than applying a
   newer replacement unconditionally to a mixed-version deployment.

## Breaking and migration-sensitive changes

### Replace deprecated core syntax

- Use canonical casts: `(bool)`, `(int)`, `(float)`, and `(string)`.
- End `case` labels with `:`, not `;`.
- Replace backtick command execution with an explicit process or
  `shell_exec()` call.
- Replace non-numeric string `++` with `str_increment()`.
- Use `''` explicitly when an empty-string array key is intended; `null`
  offsets and `array_key_exists(null, ...)` are deprecated.
- Do not derive non-CLI `argc` or `argv` from a query string. Disable
  `register_argc_argv` and validate the appropriate request input.
- Do not declare a class whose complete name is `_`.
- Replace `0 ** $negative` or `pow(0, $negative)` with `fpow()` only when IEEE
  754 behavior is deliberately required.

### Modernize lifecycle and failure handling

- Throw an exception for recoverable failure or call `exit()` for deliberate
  termination instead of `trigger_error(..., E_USER_ERROR)`.
- Implement `__serialize()` and `__unserialize()` instead of relying on
  `__sleep()`, `__wakeup()`, or `SplFixedArray::__wakeup()`.
- Ensure `__debugInfo()` returns an array.
- Do not produce output from inside a user output handler.
- Catch `Error` when code intentionally compares cyclic structures; recursive
  comparison no longer ends in an uncatchable fatal path.
- Expect fatal errors to include backtraces, including execution-time failures.
- Treat output-handler, destructor, shutdown-function, tick-handler, and
  delayed class-linking order as observable behavior and test shutdown paths.

### Audit closures, cloning, properties, and attributes

- Remove closure rebinding that attaches instances to static closures, changes
  function or method scope, unbinds `$this`, targets an unrelated object, or
  uses internal-class scope.
- In `__clone()`, reassign readonly properties directly; do not take indirect
  references to them.
- Use clone-with-properties when atomic clone-time replacement is appropriate,
  including for readonly properties.
- Attributes marking attribute classes cannot normally target abstract
  classes, enums, interfaces, or traits. Use delayed target validation only
  when runtime failure from `ReflectionAttribute::newInstance()` is acceptable.
- Account for attributes on non-class constants, property-level `#[Override]`,
  asymmetric static-property visibility, and final promoted properties.

### Make invalid-input behavior explicit

- Validate ranges, encodings, locales, signal numbers, socket parameters,
  session options, Tidy settings, and extension-specific flags before calls
  that now throw.
- Pass CSV delimiter, enclosure, and escape values deliberately; do not rely on
  the default escape argument.
- Use a valid `round()` mode and a valid `unserialize()` `allowed_classes`
  option.
- Check boolean `false` explicitly for object-returning extension APIs instead
  of assuming a resource return type.
- Do not rely on warnings, ignored arguments, fallback coercions, or `-1`
  sentinel values where the API now throws or returns `false`.

## Configuration and deployment checks

### OPcache and JIT

- Configure a JIT mode as well as a nonzero buffer; a buffer alone does not
  activate JIT.
- Treat JIT compiler initialization failure as a startup-fatal condition.
- Remove legacy entries that load OPcache as a separate shared module. Continue
  to use `opcache.enable` and `opcache.enable_cli` as needed.
- Respect the larger 64-bit maximum for
  `opcache.interned_strings_buffer` only after confirming the target build.

```ini
opcache.jit=tracing
opcache.jit_buffer_size=64M
```

### Sessions and cookies

- Use the modern two-argument `session_set_save_handler()` form.
- Make custom storage accept 32-character hexadecimal session IDs.
- Remove configuration that changes deprecated SID, cookie-only, trans-SID,
  referer, SID length, or SID alphabet settings.
- Validate that `session_start()` receives a hashmap and a compatible
  `read_and_close` value.
- Check session keys for `|` before serialization.
- Use the `partitioned` cookie option when creating partitioned cookies.

### Database code

- Replace resource checks for DBA and ODBC with explicit failure checks; their
  successful results are objects.
- Audit PDO attribute types: several driver attributes now use booleans.
- Prefer driver-specific PDO connection classes and move driver members from
  the base `PDO` class to the corresponding `Pdo\*` class.
- Remove remote `uri:` DSNs.
- Account for driver-aware placeholder parsing, DSN credential precedence,
  strict fetch-mode combinations, named constructor arguments, and
  by-reference constructor parameters.
- Recognize mysqlnd wait-timeout error `4031` when connected to a sufficiently
  recent MySQL server.

## High-value additions

### Constant expressions and declarations

Closures, first-class callables, and casts can be used in constant expressions,
including attribute arguments and property or parameter defaults. Compile-time
non-class constants can carry attributes, including `#[\Deprecated]`.

```php
const LENGTH = strlen(...);
const ZERO = (int) 0.3;

#[\Deprecated]
const LEGACY_MODE = 1;
```

### Clone-time replacement

Clone function syntax can replace selected properties during the clone
operation:

```php
$copy = clone($original, ['id' => $newId]);
```

### Exception-based filtering

Use `FILTER_THROW_ON_FAILURE` when validation failure should throw. Do not
combine it with `FILTER_NULL_ON_FAILURE`.

```php
$id = filter_var($input, FILTER_VALIDATE_INT, FILTER_THROW_ON_FAILURE);
```

### cURL request control

- Discover named cURL capabilities through `curl_version()['feature_list']`.
- Use `CURLOPT_PREREQFUNCTION` for a post-connect, pre-request decision.
- Use `CURLOPT_DEBUGFUNCTION` for request diagnostics, but not together with
  `CURLINFO_HEADER_OUT`.
- Select explicit redirect semantics with `CURLFOLLOW_OBEYCODE`,
  `CURLFOLLOW_FIRSTONLY`, or `CURLFOLLOW_ALL`.
- Use `CURLOPT_INFILESIZE_LARGE` where the older option is limited to signed
  32-bit sizes, and consider persistent share handles for safe connection
  reuse across requests.

### Data, internationalization, and document processing

- Use `Pdo\Sqlite::ATTR_TRANSACTION_MODE` to choose deferred, immediate, or
  exclusive behavior for later `beginTransaction()` calls.
- Use `IntlListFormatter` when the linked ICU supports localized AND, OR, and
  unit lists.
- Use namespace-aware XSL parameters; unqualified names take the separate
  namespace argument, while Clark notation and QNames carry it themselves.
- Treat image dimension units as data rather than assuming pixels, and account
  for HEIF/HEIC and optional SVG recognition.

## Review checklist

- [ ] Exact runtime, SAPI, extensions, drivers, and linked libraries identified.
- [ ] Deprecations enabled and reviewed in tests.
- [ ] Deprecated syntax, close calls, lifecycle hooks, and renamed APIs removed.
- [ ] Resource and internal-constant assumptions replaced with current types.
- [ ] Invalid-input and error-return paths tested.
- [ ] OPcache, JIT, session, and extension startup configuration tested.
- [ ] Database parsing, credentials, fetch modes, and transaction behavior tested.
- [ ] XML, SOAP, XSL, cURL, PCRE, locale, and encoding edge cases exercised.
