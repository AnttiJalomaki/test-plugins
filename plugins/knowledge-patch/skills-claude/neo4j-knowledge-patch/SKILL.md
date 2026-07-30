---
name: neo4j-knowledge-patch
description: Neo4j
version: 2026.06.0
license: MIT
metadata:
  author: Nevaberry
---


# Neo4j Knowledge Patch

Use this skill when upgrading, configuring, querying, integrating, or operating
Neo4j. Begin with the migration checklist, then open every reference relevant
to the database, cluster, client, import, security policy, or plugin involved.
Treat deployed configuration, schemas, application tests, and observed runtime
behavior as authoritative.

## Reference index

| Reference | Topics |
| --- | --- |
| [upgrades-deprecations-and-removals.md](references/upgrades-deprecations-and-removals.md) | Upgrade blockers, removed settings and APIs, discovery migration, store formats, platform retirement |
| [cypher-language-and-query-runtime.md](references/cypher-language-and-query-runtime.md) | Cypher selection, syntax, path matching, composition, functions, runtime corrections, graph types |
| [administration-import-backup-and-clusters.md](references/administration-import-backup-and-clusters.md) | Seeding, import, backup, copy, allocation, Fleet Manager, shell, cluster procedures |
| [security-access-and-tls.md](references/security-access-and-tls.md) | ABAC and PBAC, OIDC, privileges, TLS defaults, keys, cipher suites, security logs |
| [configuration-platforms-and-observability.md](references/configuration-platforms-and-observability.md) | Packaged defaults, logs, metrics, query APIs, CDC, Java notifications, packaging |
| [vector-search-and-genai.md](references/vector-search-and-genai.md) | Vector indexes and import, `SEARCH`, vector procedure migrations, GenAI functions |

## Breaking-change checklist

### Before upgrading

- Do not run the base 2025.06 release in production; use 2025.06.1 or later to
  avoid a checkpoint-mutex deadlock.
- Complete the discovery-v1-to-v2 transition before moving to 2025.01. Update
  discovery ports and setting names before removing migration aliases.
- Migrate `high_limit` databases offline to Block format no later than the next
  LTS. A later release cannot start or recover them through a compatibility
  path.
- Move away from the deprecated `standard` store format.
- Replace removed or renamed configuration settings, cluster tags, catch-up
  strategies, metrics, procedures, and public Java symbols.
- Confirm that hostname verification, JSON debug logging, panic behavior, log
  paths, and metrics compression changes are acceptable.
- Check the operating system, base image, and init system against the current
  support policy.

### Configuration and cluster migration

- Replace discovery port `5000` with `6000` and move the old discovery,
  Kubernetes service-port, and advertised/listen address settings to their new
  names.
- Replace server groups with server tags, including catch-up strategy names,
  leader-transfer priority, random catch-up selection, and initial server tags.
- Replace query annotation, catch-up inactivity, and maximum-database settings;
  remove settings that have no replacement.
- Expect relative log-configuration paths to resolve from
  `server.directories.configuration`.
- Replace removed discovery-migration and allocator procedures in automation.
- Keep the default `debug.log` JSON appender for supportability; add another
  appender if a second format is required.

### API and integration migration

- Use the HTTP Query API instead of the deprecated transactional HTTP API.
- Branch on GQLSTATUS codes, not error-message text.
- Stop using the server Notification API and Result Core API
  `getNotifications()`.
- Migrate old vector procedures to `SEARCH`, `CREATE VECTOR INDEX`, and
  `db.create.setNodeVectorProperty()`.
- Move `cdc.*` procedure calls to `db.cdc.*`.
- Replace removed upgrade and cluster procedures with their documented current
  commands or procedures.
- Recompile Java extensions that import removed allocator, discovery, Raft,
  transaction-memory, seeding, grouping, or query-annotation symbols.

## Cypher quick reference

### Select the language deliberately

Cypher 5 is the compatibility-focused language; Cypher 25 is the evolving
language. A database can select its language when created or altered, the DBMS
can establish the default for newly created databases, and a query can override
it:

```cypher
CYPHER 25 RETURN 1 AS value
```

The distributed configuration now selects Cypher 25 for new deployments, so do
not infer a database's language solely from historical defaults.

### Compose and shape queries

- Use `WHEN`/`ELSE` for conditional composition and `NEXT` for sequential
  composition. GQL-style braces can wrap top-level and composite queries.
- Use standalone `LET` to project variables and `FILTER` to filter mid-query.
  A write clause no longer requires a `WITH` boundary before a read clause.
- Use explicit `RETURN ALL` or `WITH ALL` to retain duplicates.
- Use `FOR` as the GQL equivalent of `UNWIND`.
- Compose supported `SHOW` and transaction-management commands with other
  Cypher statements.
- Copy entity properties with `SET target = properties(source)`, not by placing
  a node or relationship directly on the right side of `SET`.

### Match paths and labels

```cypher
MATCH REPEATABLE ELEMENTS p = (a)-[*]->(b)
RETURN p
```

- `REPEATABLE ELEMENTS` permits relationship reuse; trail semantics remains
  the default and can be requested with `MATCH DIFFERENT RELATIONSHIPS`.
- `ACYCLIC` can be combined with restrictive `ANY` and `SHORTEST` selectors.
- `IS LABELED` and `IS NOT LABELED` are GQL equivalents of label `IS` and
  `IS NOT` predicates.
- `SHORTEST` and `ANY` path patterns accept parameters.

### Account for corrected results

- `stDev()` returns `null` for empty input.
- Pipelined `COUNT(DISTINCT)` no longer overcounts when leveraged order is
  missing.
- Ordered `OR EXISTS` subqueries no longer lose a result row.
- Undirected multi-type relationship scans no longer omit sibling
  relationships.
- In Cypher 25 subquery expressions, imported variables are aggregation
  constants rather than grouping keys.

## Administration quick reference

### Make imports explicit

- Set a finite `--bad-tolerance` when malformed input must stop an import; the
  default is unlimited.
- Keep `--vector-delimiter` distinct from `--delimiter` and `--quote`.
- Use native Parquet list values for vectors when available.
- Composite import identities can retain `INTEGER` types.
- Full and incremental imports can apply their supported graph-type schema
  commands through `--schema`.
- Locate progress logs according to the installed release; their directory
  moved twice.

### Copy, backup, and seed

- Treat `--from-pagecache` on `database copy` as a whole-operation off-heap
  limit, or use the clearer `--max-off-heap-memory` spelling.
- Use `--compress` with backup-format output from database copy or import.
- Expect backup inspection order by append index, with time as the tie-breaker.
- Use provider-native credentials for cloud seeds, current seed option names,
  and `server://` artifacts for cluster-local sharded-property-database seeds.
- Use `FileSeedProvider` for filesystem locations and `CloudSeedProvider` for
  cloud seeds.

### Operate clusters and fleets

- Use the discovery command to find local servers, then bulk-register them with
  Fleet Manager when appropriate.
- Grant `SERVER MANAGEMENT` for server cordon and automatic-enable procedures.
- Treat administration `WAIT` cluster state as notifications, not result rows.
- Expect an error when revoking a privilege that cannot exist.

## Security quick reference

- Native and native linked-LDAP users can carry metadata for ABAC role rules;
  manage it with `DBMS USER METADATA MANAGEMENT`.
- Move OIDC providers from the deprecated Implicit flow to default PKCE.
- Do not place a user-defined function in a PBAC privilege.
- Invalid time functions in auth rules fail when the rule is created.
- Expect TLS hostname verification to default to enabled.
- With OpenSSL provider 3.5 or later, use `X25519MLKEM768` when hybrid
  post-quantum key exchange is required.
- Remove dependence on default CBC cipher suites and replace legacy PKCS #1 RSA
  private keys.
- Configure `s3Endpoint` for secure MinIO access in the administrative Helm
  charts.

## Vector and GenAI quick reference

### Use vector `SEARCH`

Cypher 25 vector searches accept `IN` inside the filter predicate:

```cypher
MATCH (movie:Movie)
SEARCH movie IN (
  VECTOR INDEX moviePlots
  FOR $queryVector
  WHERE movie.genre IN ['Horror', 'SciFi']
  LIMIT $topK
)
RETURN movie.title, movie.rating
```

Prefer `SEARCH` over deprecated vector-query procedures. Existing vector
indexes must be rebuilt to adopt preview Hi-Fidelity quantization and its
search expansion factor.

### Use GenAI helpers

- Configure `GENAI_AZURE_OPENAI_BASE_URL` when `ai.text` calls need a custom
  Azure OpenAI base URL.
- Use `ai.text.countToken` to estimate token count and
  `ai.text.chunkByTokenLimit` to split bounded chunks.
- Use `ai.file.embedBatch` for local or remote files; it can chunk input and
  yields an index, content, and embedding vector for every chunk.

## Working method

1. Identify the deployed server, Cypher language, edition, store format,
   operating system, plugins, and configuration provenance.
2. Read the upgrade and configuration references before replacing binaries or
   generated configuration files.
3. Review query semantics and runtime corrections before comparing result
   counts or changing application assertions.
4. Trace import, copy, backup, seeding, and cluster operations through their
   current options, paths, and output contracts.
5. Validate authentication, authorization, TLS, metrics, and logging in a
   disposable environment before rollout.
6. Test drivers, HTTP clients, Java extensions, CDC consumers, vector queries,
   and log parsers independently.
