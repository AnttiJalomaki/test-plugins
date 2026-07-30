# Cypher Language and Query Runtime

Use this reference when selecting a Cypher language, adopting GQL-aligned
syntax, updating query generators, or investigating changed result sets.

## Language selection

Neo4j provides frozen, compatibility-focused Cypher 5 alongside evolving
Cypher 25 (introduced in the 2025.06 line). Existing and new databases
historically defaulted to Cypher 5. Select Cypher 25 for a database when it is
created or altered, use `db.query.default_language` for new and initial
databases, or override one query:

```cypher
CYPHER 25 RETURN 1 AS value
```

Starting in 2026.02, the distributed `neo4j.conf` explicitly sets:

```properties
db.query.default_language=CYPHER_25
```

Consequently, a new deployment that uses the packaged file defaults newly
created databases to Cypher 25. Inspect both database state and configuration
instead of assuming the language from the database's age.

## Query composition and projection

### Conditional and sequential queries

Cypher 25 supports `WHEN`/`ELSE` branches for conditional composition and
`NEXT` for linear composition. It also accepts GQL-style curly braces around a
top-level query and around arguments to composite queries such as `UNION`,
`UNION ALL`, and conditional queries.

### `FILTER`, `LET`, and `ALL`

Use standalone `FILTER` to filter in the middle of a query and `LET` to add
projected variables:

```cypher
MATCH (p:Person)
LET name = p.name
FILTER name IS NOT NULL
RETURN name
```

A `WITH` boundary is no longer required between writing and reading clauses.
`RETURN ALL` and `WITH ALL` explicitly retain duplicates. `UNION` and
`UNION ALL` branches may return the same items in different orders; support for
that ordering is no longer deprecated.

### GQL `FOR`

Cypher 25 supports `FOR` to extract rows from a list, as the GQL equivalent of
`UNWIND` (since 2026.04.0):

```cypher
FOR item IN [1, 2, 3]
RETURN item
```

### Composable commands

Cypher 25 can combine these commands with one another and with other Cypher
statements (since 2026.05.0):

```text
SHOW INDEXES
SHOW CONSTRAINTS
SHOW CURRENT GRAPH TYPE
SHOW FUNCTIONS
SHOW PROCEDURES
SHOW SETTINGS
SHOW TRANSACTIONS
SHOW DATABASES
TERMINATE TRANSACTIONS
```

## Path matching

### Match modes

`REPEATABLE ELEMENTS` selects walk semantics, under which a relationship may
repeat within a matched path:

```cypher
MATCH REPEATABLE ELEMENTS p = (a)-[*]->(b)
RETURN p
```

Trail semantics remains the default. Request it explicitly with
`MATCH DIFFERENT RELATIONSHIPS`.

Cypher 25 also permits `ACYCLIC`, which prevents a repeated node in a path,
together with every restrictive selector below:

```text
ANY
SHORTEST
SHORTEST k
ALL SHORTEST
SHORTEST k GROUPS
```

Parameters are accepted in `SHORTEST` and `ANY` path patterns.

### Label predicates

`IS LABELED` and `IS NOT LABELED` are GQL equivalents of the existing Cypher
`IS` and `IS NOT` label predicates:

```cypher
MATCH (n)
WHERE n IS LABELED Person
RETURN n
```

## Expressions and functions

### String operations

`replace()` accepts an optional limit for the number of replacements:

```cypher
RETURN replace('banana', 'a', 'o', 2)
```

Cypher 25 also has native `string.indexOf`, `string.join`, and
`string.regexReplace` functions. The corresponding `apoc.text.*` functions are
deprecated; migrate calls to the native functions.

### Copying properties

Cypher 25 does not permit a node or relationship directly on the right side of
a `SET` properties clause. Convert the entity to a map:

```cypher
SET target = properties(source)
```

### Empty-input standard deviation

`stDev()` returns `null`, rather than `0`, for empty input.

## Graph references and identifiers

### Unified graph references

A composite constituent reference in Cypher 25 must be a single symbolic name,
such as `compdb.constituent`; do not split it into separate quoted parts such as
`` `compdb`.`constituent` ``. Resolution consistently infers whether the prefix
is a composite.

Function arguments that contain further dots use the complete string
reference:

```cypher
USE graph.byName("composite.with.dot.constituent")
```

Ambiguous database, alias, and constituent names are rejected in both Cypher
versions. A local constituent cannot be a user's home database and must be
accessed through its composite.

### Identifier and parameter characters

Cypher 25 treats U+0085 NEXT LINE as whitespace rather than allowing it in an
identifier, and rejects identifier characters that had already been
deprecated. Parameters may start with additional characters from the GQL
extended identifier character set. Update lexers and query generators.

## Constraints, schema inspection, and graph types

Cypher 25 accepts:

```cypher
SHOW NODE PROPERTY UNIQUENESS CONSTRAINTS
SHOW RELATIONSHIP UNIQUENESS CONSTRAINTS
```

`PROPERTY` is optional. The returned type names are
`NODE_PROPERTY_UNIQUENESS` and `RELATIONSHIP_PROPERTY_UNIQUENESS`.
`indexProvider` has been removed from index- and constraint-creation options.

The `propertyTypes` column returned by `db.schema.nodeTypeProperties()` and
`db.schema.relTypeProperties()` now contains Cypher type names instead of Java
type names. Update consumers that parse the result vocabulary.

`GRAPH TYPE` is generally available for production schema definition,
enforcement, and validation as of 2026.06.0. For graph-shaped inspection,
`SHOW CURRENT GRAPH TYPE AS GRAPH` returns lists of virtual nodes and
relationships instead of the string representation:

```cypher
SHOW CURRENT GRAPH TYPE AS GRAPH
```

## Concurrent transaction batches

`CALL { ... } IN CONCURRENT TRANSACTIONS` supports `DISJOINT BY`. It schedules
disjoint parallel write work before transactions begin, avoiding lock
contention and deadlocks in workloads such as merges under unique constraints
and relationship creation.

## Aggregation and runtime result corrections

### Subquery-expression grouping

Inside Cypher 25 `COLLECT`, `COUNT`, and `EXISTS` subquery expressions, imported
variables are constants rather than aggregation grouping keys. Aggregation can
therefore produce a result when the subquery matches no rows. Cypher 5 retains
the former grouping behavior for compatibility.

### Parallel Repeat planning

The parallel runtime disables its Repeat-over-VarExpand planning heuristic by
default because variable-length patterns with input cardinality one could use
excessive memory. Restore it for an individual query only when justified:

```cypher
CYPHER parallelRepeatHeuristic=enabled
MATCH (a:A {prop: 123}) ((n)-[r:R]->(m))+ (b)
RETURN a, b
```

The preparser accepts `parallelRepeatHeuristic=enabled` and
`parallelRepeatHeuristic=disabled`.

### Corrected query results

- The pipelined runtime no longer overcounts `COUNT(DISTINCT)` in plans that
  lack a leveraged order.
- The pipelined runtime no longer drops a result row when an `OR EXISTS`
  subquery contains ordering. Corrected queries may return rows that an earlier
  release omitted.
- Undirected scans across multiple relationship types no longer omit sibling
  relationships. Affected scans previously undercounted output by roughly
  half.

Re-run result-count assertions and performance tests when these operators or
patterns appear in an upgraded workload.

## Plan metadata

`EXPLAIN` and `PROFILE` output consistently reports the underlying Neo4j
version through the point release. Parsers and plan-comparison tools must
accept the more detailed version string.
