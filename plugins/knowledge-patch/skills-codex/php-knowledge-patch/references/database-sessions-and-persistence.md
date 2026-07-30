# Database, Sessions, and Persistence

Use this reference for database driver APIs, SQL parsing, connection metadata,
fetch behavior, session storage, and cookies. Batch attributions used here are
`8.4-migration`, `8.4.0`, `8.5-migration`, and `8.5.0`.

## Connection and result types

In `8.4-migration`, successful DBA and ODBC values are objects:

- DBA connections are `Dba\Connection`.
- ODBC connections are `Odbc\Connection`.
- ODBC results are `Odbc\Result`.

Replace `is_resource()` return checks with explicit failure checks such as
`=== false`.

## MySQLi and mysqlnd

### Deprecated operations

In `8.4-migration`, these MySQLi operations are deprecated:

- `mysqli_ping()` and `mysqli::ping()`;
- the kill and refresh APIs;
- explicit `mysqli_store_result()` mode; and
- compatibility constants associated with those operations.

Issue SQL `KILL` or `FLUSH` when those operations are required.

`mysqli_execute()` is deprecated in `8.5-migration`; use
`mysqli_stmt_execute()`.

### Failures and retry detection

With MySQL server 8.0.24 or newer, mysqlnd reports server wait timeouts as
error `4031`, rather than `2006` (`8.4-migration`). Retry and disconnect logic
must recognize the newer code.

Re-running the MySQLi constructor on an initialized object throws in
`8.5-migration`.

## PostgreSQL APIs and credentials

The two-argument forms of `pg_fetch_result()`, `pg_field_prtlen()`, and
`pg_field_is_null()` are deprecated in `8.4-migration`. Use their
three-argument forms with `row: null`.

PDO_PGSQL SQL written with dollar quotes should stop escaping question marks
as `??`.

Also in `8.4-migration`, PDO_PGSQL credentials embedded in the DSN override
credentials passed to the constructor. Check credential-selection assumptions
when both are present.

## Driver-specific PDO connections

Since `8.4.0`, PDO drivers can expose driver-specific subclasses. Obtain one
through `PDO::connect()` or instantiate the appropriate subclass directly so
driver functionality is represented by the concrete connection type.

Driver-aware SQL parsers are also available. In particular:

- the MySQL parser recognizes backtick identifiers and hash comments; and
- the SQLite parser recognizes backtick literals and square-bracket
  identifiers.

These parsers avoid interpreting placeholder-like text inside those constructs
as SQL parameters.

## PDO attributes and platform requirements

In `8.4-migration`, these attributes accept and return booleans instead of
integers:

- PDO_DBLIB unique-identifier and date-time attributes;
- PDO_FIREBIRD's autocommit attribute; and
- PDO_MYSQL's autocommit, emulated-prepare, and direct-query attributes.

Building PDO_FIREBIRD requires a C++ compiler and fbclient 3.0 or newer.

## PDO DSNs, constants, and methods

In `8.5-migration`:

- The remote `uri:` PDO DSN scheme is deprecated for security reasons.
- Move driver constants from `PDO` to `Pdo\Dblib`, `Pdo\Firebird`,
  `Pdo\Mysql`, `Pdo\Odbc`, `Pdo\Pgsql`, or `Pdo\Sqlite`.
- Move prefixed PostgreSQL and SQLite methods on `PDO` to the corresponding
  same-purpose driver subclass methods.
- The unusable `PDO::PGSQL_TRANSACTION_*` constants are deprecated.

Do not hard-code PDO fetch-flag integer values, because the values changed.

## PDO fetch behavior

In `8.5-migration`:

- `PDO::FETCH_CLASS` constructor arguments follow normal
  `call_user_func_array()` semantics. String keys are named arguments, and
  by-reference parameters require references in the argument array.
- Changing fetch mode during a fetch throws.
- `FETCH_PROPS_LATE` may be combined only with `FETCH_CLASS`.
- `FETCH_INTO` may not be used with `fetchAll()`.

Audit wrappers that build argument arrays or mutate statement fetch modes.

## Firebird and SQLite edge cases

In `8.5-migration`, Firebird rejects overlong cursor names. SQLite PDO rejects
NUL bytes in quoted strings and throws when a collation callback returns the
wrong type.

Since `8.5.0`, `Pdo\Sqlite::ATTR_TRANSACTION_MODE` selects the behavior used
by later `beginTransaction()` calls: deferred, immediate, or exclusive.

```php
$pdo->setAttribute(
    Pdo\Sqlite::ATTR_TRANSACTION_MODE,
    Pdo\Sqlite::TRANSACTION_MODE_IMMEDIATE,
);
$pdo->beginTransaction();
```

## ODBC validation and builds

`odbc_fetch_row()` warns and returns `false` when given a row number at or
below zero (`8.4-migration`).

In `8.5-migration`, ODBC assumes ODBC 3.5 and removes driver-specific build
flags other than DB2. On non-Windows systems, use a driver manager.

## Session handlers and identifiers

In `8.4-migration`:

- Calls to `session_set_save_handler()` with more than two arguments are
  deprecated.
- Stop changing `session.sid_length` and
  `session.sid_bits_per_character`.
- Custom session storage must accept 32-character hexadecimal identifiers.
- Stop changing deprecated settings `session.use_only_cookies`,
  `session.use_trans_sid`, `session.trans_sid_tags`,
  `session.trans_sid_hosts`, and `session.referer_check`.
- The `SID` constant is deprecated.
- A non-positive `session.gc_divisor` or negative
  `session.gc_probability` emits a warning.

## Session input validation

In `8.5-migration`:

- Serializing `$_SESSION` with a key containing `|` warns instead of silently
  failing.
- `session_start()` requires its options to be a hashmap.
- The `read_and_close` option must have a type compatible with `int`.

Validate application-composed option arrays and session keys before starting
or serializing the session.

## Partitioned cookies

Since `8.5.0`, these APIs recognize the `partitioned` cookie option:

- `session_set_cookie_params()`;
- `session_get_cookie_params()`;
- `session_start()`;
- `setcookie()`; and
- `setrawcookie()`.

```php
setcookie('sid', $value, ['partitioned' => true]);
```
