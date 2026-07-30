---
name: neo4j-knowledge-patch
description: Neo4j
version: 2026.06.0
license: MIT
metadata:
  author: Nevaberry
---


# Neo4j Knowledge Patch

Use this skill when writing Cypher, planning an upgrade, operating a DBMS or
cluster, importing data, managing schema and indexes, or maintaining Neo4j
integrations. Check the high-impact compatibility notes first, then open the
topic reference that matches the task.

## Reference index

| Reference | Topics |
| --- | --- |
| [Cypher and query behavior](references/cypher-and-query-behavior.md) | Language selection, composition, paths, functions, runtimes, administrative results |
| [Operations, observability, and packaging](references/operations-observability-and-packaging.md) | Configuration, metrics, logging, Fleet Manager, Cypher Shell, platforms |
| [Schema, indexes, import, and backup](references/schema-indexes-import-and-backup.md) | Constraints, graph types, vector indexes, seeding, import, copy, backup, store formats |
| [Security, access, and integrations](references/security-access-and-integrations.md) | TLS, OIDC, ABAC/PBAC, privileges, Query API, GenAI, Java and Helm integrations |
| [Upgrades and breaking changes](references/upgrades-and-breaking-changes.md) | Upgrade gates, discovery, renamed and removed settings, removed APIs and procedures |

## Start with the deployed reality

Before changing a query or configuration:

1. Check the Neo4j point release and edition on every relevant server.
2. Inspect the database's configured Cypher language, not only the package
   default.
3. Determine whether configuration files are retained from an upgrade or
   generated for a new installation.
4. Inventory store formats, cluster discovery state, custom Java APIs,
   monitoring series, and automation before a major upgrade.
5. Treat preview features as opt-in and verify their configuration against the
   running release.

## Breaking upgrade gates

### Avoid the base checkpoint release

Do not run production workloads on the base `2025.06` release. It can
sporadically deadlock on the checkpoint mutex; use `2025.06.1` or later.

### Finish discovery migration first

Before moving to the calendar-versioned line, complete discovery v1-to-v2
migration. Discovery v1 is removed, internal discovery traffic moves from port
`5000` to `6000`, and several discovery addresses and settings have new names.
The temporary `*.v2.*` compatibility names should not remain in final
configuration.

### Migrate legacy store formats in time

The next LTS is the last release able to read, write, or migrate `high_limit`
databases. Migrate them offline to Block format before upgrading beyond that
LTS; otherwise they will not start. Do not choose the already-deprecated
`standard` format for new stores.

### Audit removed settings and Java APIs

The calendar-versioned line removes settings for discovery v1, database
allocation, transaction-state memory, and off-heap block caches. It also
removes matching public Java settings and discovery, Raft, seeding, and
transaction-memory symbols. Replace known renamed entries and delete any
removed-without-replacement entries rather than carrying them forward.

### Expect changed defaults

New or replaced configuration files change query annotation formatting to
JSON, metrics CSV compression to ZIP, panic shutdown to enabled, and log
configuration paths. Relative log configuration paths resolve from the
configuration directory. The default `debug.log` is JSON, and TLS hostname
verification defaults to enabled.

## Deprecation migration priorities

- Move OIDC configurations from deprecated Implicit flow to the default PKCE
  flow.
- Grant `SERVER MANAGEMENT` for server-management procedures instead of
  relying on the deprecated admin-privilege behavior.
- Parse GQLSTATUS codes rather than error-message text.
- Move JSON query-log consumers from `failureReason` to `errorInfo`.
- Replace legacy PKCS #1 RSA private keys before support is removed.
- Prefer vector `SEARCH`, `CREATE VECTOR INDEX`, and
  `db.create.setNodeVectorProperty()` over retired vector procedures.
- Use `neo4j-admin backup aggregate` and `--max-off-heap-memory` in place of
  deprecated administration command forms.
- Do not place user-defined functions in PBAC privileges.

## Cypher language selection

Neo4j supports frozen, compatibility-focused Cypher 5 and evolving Cypher 25.
Select a language per database when creating or altering it, use
`db.query.default_language` for new and initial databases, or prefix one query:

```cypher
CYPHER 25
MATCH (n)
RETURN n
```

Older deployments defaulted databases to Cypher 5. Distributed configuration
from 2026.02 explicitly sets `db.query.default_language=CYPHER_25`, so new
deployments using that file default newly created databases to Cypher 25.
Always inspect the effective setting.

## High-impact Cypher 25 migrations

### Copy entity properties through a map

A node or relationship can no longer appear directly on the right side of a
`SET` properties clause:

```cypher
SET target = properties(source)
```

### Use unified composite graph names

Write a constituent as one symbolic name, such as `compdb.constituent`, rather
than separate quoted parts. Use the complete string for dotted references:

```cypher
USE graph.byName("composite.with.dot.constituent")
```

Ambiguous database, alias, and constituent names are rejected. A local
constituent cannot be a user's home database and must be accessed through its
composite.

### Update administrative-result consumers

`SHOW TRANSACTIONS` timestamps are `ZONED DATETIME`, and unavailable values in
several columns are null. `WAIT` cluster state is delivered as notifications,
not result rows. Revoking an impossible privilege raises an error. Schema
procedures now report Cypher rather than Java names in `propertyTypes`.

### Account for corrected query results

Correctness fixes can increase or decrease results compared with affected
versions:

- Pipelined `COUNT(DISTINCT)` no longer overcounts without leveraged order.
- Ordered `OR EXISTS` subqueries no longer lose a row.
- Undirected multi-type scans no longer omit sibling relationships.
- `stDev()` returns null, not zero, for empty input.

Re-baseline tests that encoded the buggy result.

## Common Cypher 25 features

### Compose and shape queries

- `WHEN`/`ELSE` composes conditional branches; `NEXT` composes linear stages.
- GQL-style braces can surround top-level and composite-query arguments.
- Standalone `LET` projects variables and `FILTER` filters mid-query.
- A `WITH` boundary is no longer required between writing and reading clauses.
- `RETURN ALL` and `WITH ALL` explicitly retain duplicates.
- Differently ordered return items across `UNION` branches are supported.
- `FOR` is the GQL equivalent of `UNWIND`.

### Control path semantics

`REPEATABLE ELEMENTS` permits relationship reuse in a walk. Trail semantics is
the default and can be explicit with `MATCH DIFFERENT RELATIONSHIPS`.
`ACYCLIC` prevents repeated nodes and works with restrictive selectors such as
`ANY`, `SHORTEST`, `ALL SHORTEST`, and their `k` variants.

### Use native operations

- `replace()` accepts an optional replacement limit.
- `SHORTEST` and `ANY` path patterns accept parameters.
- `IS LABELED` and `IS NOT LABELED` are GQL label predicates.
- `string.indexOf`, `string.join`, and `string.regexReplace` replace the
  deprecated corresponding `apoc.text.*` functions.
- Composable `SHOW` and transaction commands can mix with other statements.

## Concurrency and vector search

Use `DISJOINT BY` with `CALL { … } IN CONCURRENT TRANSACTIONS` when batches can
be partitioned before execution. It prevents lock contention and deadlocks for
disjoint writes such as unique-key merges and relationship creation.

Cypher 25 vector `SEARCH` filters accept `IN`. Prefer `SEARCH` over deprecated
vector query procedures. Hi-Fidelity Quantized Vector Search is a preview
index option that reranks expanded quantized results with full-precision
vectors; existing indexes must be rebuilt to adopt it.

## Import and backup safety

- Set an explicit finite `--bad-tolerance` if malformed input should stop an
  import; full and incremental import otherwise default to unlimited tolerance.
- Keep `--vector-delimiter` distinct from both `--delimiter` and `--quote`.
- `database copy --from-pagecache` limits off-heap memory for the entire copy;
  prefer the clearer `--max-off-heap-memory`.
- `database copy` and `database import` can compress backup-format output with
  `--compress --target-format=backup`.
- Backup inspection is ordered by append index, then time for ties.
- Use `neo4j-admin backup aggregate`, not the deprecated
  `database aggregate-backup`.

## Observability checks

Do not depend on `cluster.internal.*` being collected by default. Default
metrics use `neo4j.count` instead of `ids_in_use`; the store-size series is
`<prefix>.store.size.full`. Old causal-clustering and discovery-v1 metrics are
removed or moved, and several Raft cache/retry metrics are deprecated.

Query logs will eventually default to JSON for new installations after the
next LTS. Retained `server-logs.xml` preserves an upgrade's existing format.
JSON is richer but larger, so size log storage and update parsers before
switching.

## When to open each reference

- For query syntax, semantics, result changes, planners, or runtimes, open
  [Cypher and query behavior](references/cypher-and-query-behavior.md).
- For server settings, metrics, logs, Fleet Manager, shell behavior, or
  supported platforms, open
  [Operations, observability, and packaging](references/operations-observability-and-packaging.md).
- For constraints, graph types, vectors, seeding, imports, backups, or store
  formats, open
  [Schema, indexes, import, and backup](references/schema-indexes-import-and-backup.md).
- For authentication, authorization, TLS, APIs, plugins, or client behavior,
  open
  [Security, access, and integrations](references/security-access-and-integrations.md).
- For a major-version readiness review, removed configuration, public Java
  surface, discovery, or procedure migrations, open
  [Upgrades and breaking changes](references/upgrades-and-breaking-changes.md).
