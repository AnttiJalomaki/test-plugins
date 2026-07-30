# Cypher and Query Behavior

## Choose the query language deliberately

Neo4j provides frozen, compatibility-focused Cypher 5 alongside evolving
Cypher 25 (2025.06). Existing and new databases in that release default to
Cypher 5. Select Cypher 25 when creating or altering a database, set
`db.query.default_language` for new and initial databases, or override a query:

```cypher
CYPHER 25 RETURN 1 AS value
```

Starting in 2026.02, the distributed `neo4j.conf` explicitly sets
`db.query.default_language=CYPHER_25`. A new deployment using the packaged
configuration therefore creates databases with Cypher 25 by default, while an
older or retained configuration can behave differently. Inspect the effective
database and configuration settings.

## Compose queries and intermediate results

Cypher 25 adds `WHEN`/`ELSE` for conditional query composition and `NEXT` for
linear composition (2025.06). GQL-style braces may wrap top-level queries and
the arguments of composite queries, including `UNION`, `UNION ALL`, and
conditional queries.

Standalone `LET` adds projected variables and `FILTER` filters mid-query. A
`WITH` boundary is no longer required between writing and reading clauses:

```cypher
MATCH (p:Person)
LET name = p.name
FILTER name IS NOT NULL
RETURN name
```

`RETURN ALL` and `WITH ALL` explicitly retain duplicates. `UNION` and
`UNION ALL` branches with differently ordered return items are supported and
are no longer deprecated.

Cypher 25 also supports GQL `FOR` as the equivalent of `UNWIND` (2026.04.0):

```cypher
FOR item IN [1, 2, 3]
RETURN item
```

Composable commands can be combined in one query and mixed with other Cypher
statements (2026.05.0): `SHOW INDEXES`, `SHOW CONSTRAINTS`,
`SHOW CURRENT GRAPH TYPE`, `SHOW FUNCTIONS`, `SHOW PROCEDURES`,
`SHOW SETTINGS`, `SHOW TRANSACTIONS`, `SHOW DATABASES`, and
`TERMINATE TRANSACTIONS`.

## Select path semantics

Cypher 25 adds `REPEATABLE ELEMENTS` walk semantics, in which a relationship
may repeat in a matched path (2025.06). Trail semantics remains the default and
can be requested explicitly with `MATCH DIFFERENT RELATIONSHIPS`:

```cypher
MATCH REPEATABLE ELEMENTS p = (a)-[*]->(b)
RETURN p
```

`ACYCLIC` prevents repeated nodes and can be combined with `ANY`, `SHORTEST`,
`SHORTEST k`, `ALL SHORTEST`, and `SHORTEST k GROUPS` (2026.05.0).
Parameters are also accepted in `SHORTEST` and `ANY` path patterns (2025.06).

## Functions, predicates, and identifiers

The four-argument `replace()` form limits the number of replacements (2025.06):

```cypher
RETURN replace('banana', 'a', 'o', 2)
```

GQL `IS LABELED` and `IS NOT LABELED` are equivalents of Cypher `IS` and
`IS NOT` label predicates (2026.04.0):

```cypher
MATCH (n)
WHERE n IS LABELED Person
RETURN n
```

Native `string.indexOf`, `string.join`, and `string.regexReplace` are available
(2026.05.0). Migrate from the corresponding deprecated `apoc.text.*`
functions.

Cypher 25 treats U+0085 NEXT LINE as whitespace instead of allowing it in
identifiers and rejects formerly deprecated identifier characters (2025.06).
Parameters may start with additional characters from the GQL extended
identifier character set.

## Property and graph-reference compatibility

Cypher 25 no longer accepts a node or relationship directly on the right side
of a `SET` properties clause (2025.06). Convert the entity to a map:

```cypher
SET target = properties(source)
```

A composite constituent reference must be one symbolic name, such as
`compdb.constituent`, rather than separate quoted parts such as
`` `compdb`.`constituent` ``. Resolution infers whether the prefix is a
composite. In a function argument, pass a dotted name as a complete string:

```cypher
USE graph.byName("composite.with.dot.constituent")
```

Ambiguous database, alias, and constituent names are rejected in both language
versions. A local constituent cannot be a user's home database and must be
accessed through its composite.

## Aggregation semantics and corrected results

Inside Cypher 25 `COLLECT`, `COUNT`, and `EXISTS` subquery expressions,
imported variables are constants rather than aggregation grouping keys
(2025.06). Aggregation can therefore produce a result when no rows match.
Cypher 5 retains the former grouping behavior.

The pipelined runtime no longer overcounts `COUNT(DISTINCT)` when a plan lacks
a leveraged order, and `stDev()` now returns null rather than zero for empty
input (2026.05.0).

Two further correctness fixes landed in 2026.06.0:

- An ordered `OR EXISTS` subquery no longer silently drops a result row.
- An undirected scan over multiple relationship types no longer omits sibling
  relationships; affected scans previously undercounted by roughly half.

Re-baseline assertions that captured the previous incorrect results.

## Runtime planning

The parallel runtime disables the Repeat-over-VarExpand planning heuristic by
default because a variable-length pattern with input cardinality one can use
excessive memory (2026.05.0). Restore it for one query only when justified:

```cypher
CYPHER parallelRepeatHeuristic=enabled
MATCH (a:A {prop: 123}) ((n)-[r:R]->(m))+ (b)
RETURN a, b
```

The preparser option accepts `enabled` and `disabled`.

`EXPLAIN` and `PROFILE` consistently include the Neo4j point release in plan
output (2026.04.0). Plan parsers and comparisons must accept the more detailed
version.

## Concurrent transaction batches

`CALL { … } IN CONCURRENT TRANSACTIONS` supports `DISJOINT BY` (2026.06.0).
It schedules disjoint parallel write work before transactions begin, avoiding
lock contention and deadlocks in operations such as merges under unique
constraints and relationship creation.

## Vector search syntax

Cypher 25 vector `SEARCH` accepts `IN` in its filter predicate (2026.06.0):

```cypher
MATCH (movie:Movie)
SEARCH movie IN (
  VECTOR INDEX moviePlots
  FOR $queryVector
  WHERE movie.genre IN ['Horror', 'SciFi']
  LIMIT $topK
)
RETURN movie.title AS title, movie.rating AS rating
```

Prefer `SEARCH` over the deprecated `db.index.vector.queryNodes()` and
`db.index.vector.queryRelationships()` procedures.

## Administrative and schema result contracts

In Cypher 25, `SHOW TRANSACTIONS` returns `startTime` and
`currentQueryStartTime` as `ZONED DATETIME`, not `STRING` (2025.06).
Unavailable values in several transaction columns are null. Administration
commands using `WAIT` report cluster state as notifications rather than result
rows, and revoking a privilege that cannot exist raises an error.

The `propertyTypes` column returned by `db.schema.nodeTypeProperties()` and
`db.schema.relTypeProperties()` contains Cypher type names instead of Java type
names. Update consumers that parse these rows.

In Cypher 25 from 2025.12, `dbms.setDefaultAllocationNumbers()` accepts
`propertyShardReplicas`, and `dbms.showTopologyGraphConfig()` returns the same
field.
