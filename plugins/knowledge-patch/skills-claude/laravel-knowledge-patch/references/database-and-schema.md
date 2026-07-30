# Database, Query Builder, Schema, and Transactions

## Contents

- [Connections, constructors, and inspection](#connections-constructors-and-inspection)
- [Schema columns and indexes](#schema-columns-and-indexes)
- [Query construction](#query-construction)
- [Writes, transactions, and destructive operations](#writes-transactions-and-destructive-operations)
- [Driver tooling and transport security](#driver-tooling-and-transport-security)

## Connections, constructors, and inspection

- **Multi-schema inspection** `[12.0-upgrade]`: `Schema::getTables()`,
  `getViews()`, and `getTypes()` inspect every schema by default and accept a
  string or array `schema:` filter. `getTableListing()` returns qualified names
  unless `schemaQualified: false` is passed. `db:table` and `db:show` likewise
  include every schema on MySQL, MariaDB, and SQLite.
- **Low-level constructors** `[12.0-upgrade]`: construct `Blueprint` and
  `Grammar` with a `Connection` first. `Grammar::setConnection()` and
  `Connection::withTablePrefix()` are removed; use
  `$connection->getTablePrefix()`. `Blueprint::getPrefix()` and the grammar
  table-prefix getters and setters are deprecated.
- **Native MariaDB CLI** `[12.0.0]`: database CLI operations use MariaDB-native
  commands; install the corresponding binaries.
- **PDO fetch modes** `[12.0.0]`: query-builder retrieval operations can select
  a PDO fetch mode.
- **Enum database connections** `[2025-09]`: database manager methods and model
  connection properties and methods accept enum selectors.
- **Database reconnection events** `[2025-12]`: reconnections dispatch
  `ConnectionEstablished`; `SSL error: unexpected eof` is treated as a lost
  connection.
- **SQLite transaction modes** `[2025-08]`: configure `DEFERRED`, `IMMEDIATE`,
  or `EXCLUSIVE` with `transaction_mode`.
- **SQLite arbitrary pragmas** `[2025-09]`: configure additional pragmas
  additively; existing option locations remain valid after the attempted move
  was reverted.
- **SQLite URI connections** `[2026-05]`: database names may use the `file:`
  URI form with URI connection options.
- **PostgreSQL transaction poolers** `[2026-06]`: connections support
  transaction-pooler operation.

## Schema columns and indexes

- **SQLite JSON and JSONB** `[2025-03]`: `$table->json()` and
  `$table->jsonb()` create the supported native SQLite types.
- **PostgreSQL nulls-not-distinct** `[2025-03]`: add
  `->nullsNotDistinct()` to a unique index to treat null values as equal.
- **MariaDB UUID fallback** `[2025-04]`: UUID columns use `char(36)` before
  MariaDB 10.7, where the native UUID type is unavailable.
- **Current date and year defaults** `[2025-05]`: `date` and `year` columns
  support `useCurrent()`.
- **Positioned morph columns** `[2025-08]`: `morphs()` and
  `nullableMorphs()` accept `after:` to position both generated columns.
- **Online indexes** `[2025-08]`: `->online()` emits concurrent index creation
  on PostgreSQL and online creation on SQL Server.
- **PostgreSQL virtual columns** `[2025-10]`: virtual columns are supported by
  the PostgreSQL schema grammar.
- **Instant column additions** `[2025-11]`: request instant column addition
  where a compatible database can avoid a table rebuild.
- **MySQL DDL locking** `[2026-01]`: schema changes can select supported MySQL
  DDL lock behavior.
- **PostgreSQL `tsvector` columns** `[2026-03-laravel-12]`: create native
  search vectors with `$table->tsvector('search_document')`.
- **MariaDB vector indexes** `[2026-06]`: use Laravel schema APIs for MariaDB
  vector indexes.
- **PostgreSQL conversions** `[2026-07]`: use
  `->using($expression)->change()` to supply the conversion expression for a
  changed column.

## Query construction

- **Literal values between columns** `[2025-07]`: use
  `whereValueBetween($value, [$lowerColumn, $upperColumn])`.
- **Subquery range bounds** `[2026-01]`: query-builder `between` variants accept
  subqueries for values and for column-bound ranges.
- **MySQL query timeouts** `[2026-02]`: apply `timeout()` to an individual
  MySQL query.
- **`DatePeriod` ranges** `[2026-02]`: `whereBetween()` accepts `DatePeriod`,
  including periods with no end date.
- **Eloquent subqueries in updates** `[2026-02]`: update values may be Eloquent
  builders or relationship queries without conversion to base builders.
- **Precomputed PostgreSQL vectors** `[2026-02]`: `whereFullText()` accepts a
  vector option for an existing `tsvector` column.
- **Caller-defined ordering** `[2026-03-laravel-12]`: use
  `inOrderOf($column, $values)` to order rows by a supplied value sequence.
- **Touching multiple columns** `[2026-03-laravel-12]`: `touch()` accepts an
  array of timestamp columns.
- **Semantic vector queries** `[13.0.0]`: PostgreSQL with `pgvector` supports
  similarity queries such as `whereVectorSimilarTo()`, including embedding a
  plain-language query.
- **Sort direction enums** `[2026-05]`: ordering APIs accept the
  `SortDirection` enum.

## Writes, transactions, and destructive operations

- **Conditional migrations** `[2025-03]`: define `shouldRun(): bool` on a
  migration for runtime gating.
- **Prohibitable seeding** `[2025-04]`: `DB::prohibitDestructiveCommands()` can
  block `db:seed`.
- **Transaction rollback callbacks** `[2025-09]`: register rollback-only work
  with `DB::afterRollback()`.
- **Prohibitable schema dumps** `[2025-11]`: schema dumping participates in
  destructive-command prohibition.
- **Clean deadlock retries** `[2026-02]`: Laravel rolls back a lingering PDO
  transaction before retrying a commit deadlock.
- **Non-empty upsert keys** `[13.0-upgrade]`: MySQL and MariaDB `upsert()` throw
  `InvalidArgumentException` when `uniqueBy` is empty.
- **Joined MySQL deletes** `[13.0-upgrade]`: requested `ORDER BY` and `LIMIT`
  are emitted. Unsupported database variants raise `QueryException` instead of
  silently executing an unbounded delete.
- **Schema dump without migration data** `[2026-06]`: pass
  `schema:dump --without-migration-data` to omit migration records.
- **Collision-free migration timestamps** `[2026-07]`: `make:migration`
  generates ordered, collision-free prefixes for closely timed migrations.

## Driver tooling and transport security

- **Disable MySQL schema SSL** `[2025-05]`: migration squash and restore can
  explicitly disable SSL where client defaults request unavailable TLS.
- **MySQL schema TLS credentials** `[2026-02]`: dump and load commands accept
  configured SSL certificates and keys.
- **Mode-less PostgreSQL full text** `[2025-11]`: issue PostgreSQL full-text
  queries without naming a search mode.
