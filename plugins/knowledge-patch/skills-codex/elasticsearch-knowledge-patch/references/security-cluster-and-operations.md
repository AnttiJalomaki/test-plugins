# Security, cluster APIs, and operations

Use this reference for authentication, federation, secure settings, node
runtime changes, cross-project access, and operational diagnostics.

## Credentials, keys, and authorization

### API-key hashes and secure settings

In 9.0.0, API-key credential hashes can use `SSHA-256`. Secure settings are no
longer accepted in YAML and must be supplied through Elasticsearch's
secure-settings mechanism.

### Connector deletion and privileges

In 9.0.0, Connector APIs add soft and hard deletion through a delete URL
parameter, and managed connector indices must use the required prefix.

### Extensible security and federation

In 9.1.0, `SecurityExtensions` can provide a custom
`ServiceAccountTokenStore`; the SAML identity provider supports custom
attributes; and JWT access tokens may use the `at+jwt` `typ` header.
Elasticsearch also adds a Microsoft Graph delegated-authorization realm
plugin.

### Cross-cluster keys and TLS extensibility

In 9.2.0, cross-cluster API keys gain signing and trust configuration, and the
transport TLS handshake timeout is adjustable. TLS extensions can implement
the `SslProfileExtension` SPI and receive reload notifications through an
`SslProfile` listener.

### Security and federation updates

In 9.3.0, JWT realms can periodically reload PKC JWK sets, successful SAML
responses include in-response-to metadata, and cross-cluster API keys carry
and validate certificate identities. Secure-settings reload responses include
secure-setting names and the keystore modification time.

### API keys and service accounts

In 9.4.0, API keys can be cloned through a dedicated endpoint, and
service-account-token APIs are available in Serverless. The `read` index
privilege consistently authorizes cross-cluster search regardless of
`ccs_minimize_roundtrips`.

## SAML configuration

In 9.2.0, SAML private attributes are configurable. URL-based SAML metadata
resolution has configurable HTTP read and connect timeouts.

## Security statistics

In 9.2.0, `/_security/stats` exposes document-level-security statistics,
including DLS cache usage, hits, misses, and timing.

## Cluster resolution and allocation

### Cluster-only resolution

In 9.0.0, `_resolve/cluster` can query cluster information without an index
expression and accepts a user-configurable timeout.

### Per-tier balancing

In 9.1.0, shard-allocation balancing weights can be set independently per data
tier.

### Synonym refresh control

In 9.1.0, synonyms PUT and delete APIs accept `refresh`, which waits for
updated synonyms to become accessible and reloads affected analyzers.

## Cross-cluster and cross-project operation

### Cross-project routing

In 9.3.0, cross-project search and `project_routing` extend to `_search`,
`_async_search`, `_msearch`, EQL, field capabilities, SQL, and JDBC.
Point-in-time creation and closure can span projects. Cross-project searches
default to minimizing round trips.

Stateful cross-cluster use in 9.3.0 disables `_delete_by_query` and
`_update_by_query`.

In 9.4.0, project routing extends to templated searches, data streams, scrolls,
and the SQL CLI. The SQL CLI and JDBC client can authenticate with API keys.

## Runtime and packaging

### Java, Lucene, and containers

In 9.0.0, Elasticsearch bundles JDK 24, upgrades to Lucene 10.1.0, and bases
its default Docker image on UBI minimal rather than Ubuntu. Startup ignores
`_JAVA_OPTIONS`.

Elasticsearch Entitlements permanently replace the Java SecurityManager,
which Java 24 disables.

In 9.4.0, Elasticsearch upgrades to Apache Lucene 10.4. The
`cloud-ess-fips` package defaults to FIPS 140-3.

### File-backed reloads

In 9.1.0, configuration reload detects Kubernetes CSI-style `..data` symlink
switches. TLS reload watches SSL files rather than their containing
directories.

## Limits, defaults, and telemetry

### Mustache output limits

In 9.0.0, `mustache.max_output_size_bytes` limits Mustache script result
length.

### Allocation and reindex metrics

In 9.0.0, `replica_unassigned_buffer_time` increases from three to five
seconds. Reindexing metrics report seconds rather than milliseconds.

### Circuit breakers, shard capacity, and TLS

In 9.3.0, cat APIs add a circuit-breakers endpoint, shard-capacity health
thresholds become configurable, and Elasticsearch can limit concurrent TLS
handshakes.

## Search diagnostics and asynchronous operations

In 9.4.0, query logging covers `_search`, ES|QL, EQL, and SQL. A search-task
watchdog can log hot threads for slow searches. Async result retrieval adds
`return_intermediate_results` to control delivery of in-progress partial
results, and async task status exposes `keep_alive`.

## Shutdown and task operation

In 9.4.0, persistent-task reassignment during node shutdown becomes opt-in, and
shutdown status reports shard snapshot pauses.
