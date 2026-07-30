# Schema, Indexes, Import, and Backup

## Constraint and schema inspection

Cypher 25 accepts
`SHOW [NODE|RELATIONSHIP] PROPERTY UNIQUENESS CONSTRAINTS`, with `PROPERTY`
optional (2025.06). Returned type names are `NODE_PROPERTY_UNIQUENESS` and
`RELATIONSHIP_PROPERTY_UNIQUENESS`. The `indexProvider` option is removed from
index and constraint creation commands.

The `propertyTypes` column from `db.schema.nodeTypeProperties()` and
`db.schema.relTypeProperties()` now uses Cypher type names rather than Java
type names. Update any schema tooling that expects Java vocabulary.

## Graph types

Full import accepts the preview `ALTER CURRENT GRAPH TYPE SET {…}` through
`neo4j-admin database import full --schema` (2026.05.0).

`GRAPH TYPE` is generally available for production schema definition,
enforcement, and validation (2026.06.0). Inspection can return graph-shaped
data:

```cypher
SHOW CURRENT GRAPH TYPE AS GRAPH
```

This form returns lists of virtual nodes and relationships rather than a
string. Incremental import also accepts
`ALTER CURRENT GRAPH TYPE ADD/DROP/ALTER {…}` through `--schema`, extending
schema changes beyond the full-import path.

## Vector indexes and search migration

Preview Hi-Fidelity Quantized Vector Search expands a search over quantized
vectors and reranks with full-precision vectors (2026.06.0). Enable its
quantization type and default expansion factor per index:

```cypher
CREATE VECTOR INDEX moviePlots IF NOT EXISTS
FOR (m:Movie)
ON m.embedding
OPTIONS {indexConfig: {
  `vector.quantization.type`: 'binary',
  `vector.default_search_expansion_factor`: 2.0,
  `vector.dimensions`: 1536,
  `vector.similarity_function`: 'cosine'
}}
```

An existing index must be rebuilt to use HFQ.

In Cypher 25, `db.index.vector.queryNodes()` and
`db.index.vector.queryRelationships()` are deprecated in favor of `SEARCH`.
`db.index.vector.createNodeIndex()` and `db.create.setVectorProperty()` are
removed; use `CREATE VECTOR INDEX` and `db.create.setNodeVectorProperty()`.

## Import parsing and identity

Multi-column identities in `neo4j-admin import` may use `INTEGER` ID types
instead of being forced to `STRING` (2026.04.0).

For vector imports, `--vector-delimiter` must differ from both `--delimiter`
and `--quote` (2026.06.0). The importer can read vector values directly from
native Parquet list types.

From 2025.12, full and incremental import default `--bad-tolerance` to `-1`,
meaning unlimited rather than `1000`. Supply a finite value when the operation
must stop after a bounded number of bad entries.

## Copy and backup output

`neo4j-admin database copy` and `neo4j-admin database import` accept
`--compress` when producing backup-format output with
`--target-format=backup` (2026.04.0).

`neo4j-admin backup inspect` orders results by append index and uses time to
order entries with the same append index. Consumers should depend on this
ordering contract.

From 2025.01, `neo4j-admin database copy --from-pagecache=<size>` limits
off-heap memory for the entire copy operation, covering reads and writes, not
just the source read cache. Prefer the clearer
`--max-off-heap-memory=<size>` name.

Replace deprecated `database aggregate-backup` with
`neo4j-admin backup aggregate`. For `neo4j-admin database migrate`, replace
deprecated `--page-cache` with `--max-off-heap-memory`.

From 2025.10, backup metadata can be limited to named users, including only
those users and their role assignments:

```text
--include-metadata=users=alice,bob
```

## Database seeds

Cypher 25 removes `seedCredentials`; cloud-seed authentication comes from the
cloud provider's built-in mechanism (2025.06). It also:

- replaces `existingDataSeedInstance` with `existingDataSeedServer`;
- adds `seedSourceDatabase` to filter restored backup artifacts;
- deprecates the now-optional `existingData` option; and
- accepts Java `Long` as well as `Int` parameters in `CREATE DATABASE`.

A sharded property database can seed from artifacts in cluster members'
repository folders by supplying `server://` values in `seedUri` (2026.04.0):

```cypher
CREATE DATABASE spd OPTIONS {
  seedUri: ["server://server-1/", "server://server-2/"]
}
```

`S3SeedProvider` is replaced by `CloudSeedProvider` from 5.26.
`URLConnectionSeedProvider` no longer handles `file` locations in either
Cypher 5 or Cypher 25; use `FileSeedProvider` for filesystem seeds.

## Store-format deadlines

The next LTS is the final release that can read, write, or migrate
`high_limit` databases (2026.06.0). Before upgrading beyond it, migrate these
databases offline to Block format. A remaining `high_limit` database will fail
to start and has no compatibility fallback.

The `standard` store format has been deprecated since 5.23. Avoid it for new
databases and plan to move existing stores away from it.
