# PPL and SQL

Use this reference for Calcite routing, PPL syntax and result semantics, SQL
pagination, unified query APIs, grammar tooling, and PPL Alerting.

## Engine selection and common behavior

### Calcite-backed PPL

Calcite is enabled by default in 3.0.0. Its PPL path supports `join`, `lookup`,
`IN`, relation, `exists`, and scalar subqueries; `BETWEEN`, `dedup`, `parse`,
JSON casts, `CASE`, and `TYPEOF`; multiple index patterns, nested fields,
comments, and additional scalar and data types.

### PPL and SQL behavior

Since 3.2.0, failed Calcite PPL queries do not fall back to the v2 engine by
default. Calcite adds `compare_ip`, IP casts, function-argument coercion,
improved date comparisons, and broader expression support. Date and time
functions default to UTC in both PPL and SQL.

### PPL query routing and data sources

Since 3.3.0, unsupported Calcite PPL commands implicitly fall back to v2 even
though general failures do not. Direct-query data sources are supported, and
PIT searches use `_shard_doc` as their default sort tiebreaker.

### Unified query APIs

Since 3.6.0, SQL has a unified query parser API, unified-API profiling, and
native Calcite SQL planning. PPL requests can be cancelled through
`_tasks/_cancel`, accept `fetch_size`, and expose a grammar-bundle API for
third-party query tooling.

### SQL pagination defaults

OpenSearch 3.0 removes `plugins.sql.pagination.api`, deprecates Scroll-based SQL
pagination, and defaults to Point in Time. Deprecated OpenDistro endpoints and
legacy `opendistro`-prefixed settings are removed.

## Fields, metadata, and schemas

### PPL metadata fields

Since 2.19.0, PPL queries can reference metadata fields.

### PPL pipeline expansion

The 3.1.0 Calcite path adds `flatten`, `expand`, `trendline`, `appendcol`,
`grok`, `top`, `rare`, `fillnull`, `describe`, and `eventstats`. It supports
`match_only_text` and merges object fields while discovering an index schema.

### PPL traversal and field commands

Since 3.6.0, PPL supports bidirectional `graphlookup` with literal starting
values, `convert` with five conversion functions, `mvexpand`, `nomv`, and
`fieldformat`. It also supports result highlighting, automatic extraction for
`spath`, the `contains` operator, and trailing or empty pipes.

## Commands and pipeline syntax

### PPL command expansion

Since 3.3.0, PPL adds `spath`, `rex`, and `regex`; `rex` supports `sed` mode and
`offset_field`. It also adds `append`, expands `join` with field lists and
options, adds time modifiers to `search`, supports wildcard `rename`, and adds
Calcite `timechart`.

### PPL commands and multivalue functions

Since 3.4.0, Calcite PPL adds `chart`, `streamstats`, `multisearch`, `replace`,
and `appendpipe`, plus `mvdedup`, `mvindex`, and `mvappend`. `timechart` adds
per-second, per-minute, per-hour, and per-day functions. `replace` supports
wildcards, its eval form supports regular expressions, and `regexp_replace()`
is an alias.

### New PPL commands and multivalue functions

Since 3.5.0, PPL adds `addtotals`, `addcoltotals`, `transpose`, and `mvcombine`;
the `tonumber`, `mvzip`, `split`, `mvfind`, and `mvmap` eval functions; and
`category_field` for the `ml` command.

### PPL syntax expansion

Since 3.7.0, `union` uses type coercion and UNION ALL semantics. `convert` adds
`ctime`, `mktime`, `mstime`, `dur2sec`, and `timeformat`. Predicates accept
`IS [NOT] NULL`, and `head` and `top` accept `limit=N`.

## Functions, expressions, and limits

### PPL functions and expressions

Since 3.1.0, PPL supports `DISTINCT_COUNT_APPROX`, `earliest`, `latest`,
`coalesce`, `isempty`, `isblank`, `ispresent`, `geoip`, `cidrmatch`, JSON
functions, cryptographic hashes, lambdas, array functions, and decimal
literals. `query.size_limit` limits final results rather than intermediate
processing.

### PPL functions and result behavior

Since 3.3.0, PPL supports `values` and `list` statistics; `first` and `last`
aggregates; `mvjoin`, `strftime`, and `regex_match`; nonnumeric and eval-context
`max`/`min`; ISO 8601 strings; and `distinct_count`, `earliest`, and `latest` in
`eventstats`. `count(*)` and `dc` cap at `MAX_INTEGER`, decimal `mod` returns a
decimal, and an unmatched index pattern raises `IndexNotFoundException`.

### PPL aggregation and time controls

Since 3.4.0, `streamstats` and `eventstats` accept `bucket_nullable`; `top` and
`rare` accept `usenull`; and `timechart` selects its timestamp with `timefield`.
Span expressions support milliseconds, decimal literals, and an implicit
`@timestamp`.

### PPL expression and request behavior

Since 3.4.0, `rex`, `spath`, and `parse` extractions automatically convert
types, `geoip` accepts IP input, eval division returns decimal, and `like` has
an optional `case_sensitive` argument. Calcite supports query timeouts, Explain
can return YAML, and subsearch and join limits are configurable. A zero or
negative `subsearch.maxout` means unlimited.

### PPL execution behavior and diagnostics

Since 3.5.0, when `plugins.ppl.syntax.legacy.preferred=false`, `join` defaults
to `max=1`. `lookup` accepts `OUTPUT` as an alias for `REPLACE`, `spath`
supports dynamic fields, and profiling reports phase- and operator-level
metrics.

### PPL result and integration behavior

Since 3.6.0, final struct values return as maps rather than lists. A missing or
null `JSON_EXTRACT` path returns null, as does arithmetic that overflows a
double to infinity. `FIRST`, `LAST`, and `TAKE` accept text fields and scripts.
PPL can create or update Prometheus rules.

### Calcite query behavior

Since 3.7.0:

- `NOT IN` and `NOT LIKE` exclude null and missing values.
- `COALESCE(null, integer)` retains an integer type.
- PPL wildcard searches no longer drop documents when a field is text in one
  index and keyword in another.
- Calcite `dedup` preserves sort order.
- Dotted-path `eval` assignments preserve their map root.
- `json_set` and `json_delete` handle `$.key` paths.
- Under fine-grained access control, SQL cursor continuation stays within the
  original query indexes.

## SQL and unified query features

### SQL vector and unified-query support

Since 3.7.0, SQL provides the `vectorSearch()` table function with k-NN
pushdown and filtering modes. The query-only unified V2 path rejects DML and
DDL and supports joins, `IN` and `EXISTS` subqueries, derived tables, window
functions, `LIMIT`/`OFFSET`, and `LENGTH`, `REGEXP_REPLACE`, and `DATE_TRUNC`.

## PPL monitors and Alerting

### PPL Alerting operations

Since 3.4.0, PPL Alerting supports monitor execution and statistics,
get/search/delete monitor operations, and alert retrieval and lifecycle
operations. Alerting V2 roles are present in `roles.yml`, and Dashboards
bucket-level monitor triggers can carry keyword filters.

### PPL Alerting transition

In 3.6.0, experimental PPL Alerting assets are removed pending refactoring. The
Dashboards PPL Alerting APIs move to v1 endpoints and no longer keep separate
legacy and PPL paths.

### PPL monitor operations

Since 3.7.0, Alerting exposes PPL monitor CRUD and manual execution, with RBAC
checks for manual runs. PPL monitor names can contain up to 100 characters
rather than 30.
