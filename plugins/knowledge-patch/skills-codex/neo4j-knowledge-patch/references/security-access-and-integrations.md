# Security, Access, and Integrations

## Attribute- and property-based access control

ABAC applies to native users and native linked LDAP users as well as externally
authenticated SSO users (2026.06.0). Administrators can tag native DBMS users
and reference those tags in ABAC rules for dynamic role assignment. Managing
the tags requires `DBMS USER METADATA MANAGEMENT`.

A user-defined function can no longer be defined as part of a PBAC privilege
(2026.06.0). This unsupported combination did not enforce the behavior implied
by its definition; remove it from privilege rules.

## OIDC and authentication rules

`dbms.security.oidc.<provider>.auth_flow` supports PKCE and Implicit, with PKCE
as the default (2026.06.0). Implicit is deprecated and will be removed; migrate
configuration to PKCE.

The OIDC settings `dbms.security.oidc.<provider>.auth_params` and
`dbms.security.oidc.<provider>.client_id` are deprecated.

Creating an auth rule with an invalid time function now fails immediately
rather than deferring the failure to authorization-time evaluation
(2026.05.0). Treat creation-time rejection as configuration validation.

## Privilege changes

`dbms.cluster.cordonServer()`,
`dbms.cluster.setAutomaticallyEnableFreeServers()`, and
`dbms.cluster.uncordonServer()` require `SERVER MANAGEMENT`. Relying on an
admin privilege to call them is deprecated; grant the specific privilege.

In Cypher 25, revoking a privilege that cannot exist raises an error
(2025.06). Administrative automation must handle the failure.

## TLS defaults and key lifecycle

With OpenSSL provider 3.5 or later, TLS can use the
`X25519MLKEM768` hybrid key exchange, combining X25519 and ML-KEM-768 for
post-quantum protection (2026.05.0).

From 2025.10, four insecure Java 21 CBC suites leave the defaults, although
they remain explicitly configurable:

```text
TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384
TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256
TLS_DHE_RSA_WITH_AES_256_CBC_SHA256
TLS_DHE_RSA_WITH_AES_128_CBC_SHA256
```

`dbms.ssl.policy.*.verify_hostname` defaults to `true` rather than `false`.
After an upgrade, TLS policies verify peer hostnames unless existing
configuration explicitly fixes the value.

Neo4j still loads PKCS #1 keys with a
`-----BEGIN RSA PRIVATE KEY-----` header, but this support is deprecated and
will be removed. Replace legacy server keys.

## Query and transactional HTTP APIs

Query API transaction identifiers are six characters rather than four
(2026.04.0). Integrations that validate or store them must allow the longer
value.

The transactional HTTP API is deprecated in 5.26 in favor of the HTTP Query
API, which is enabled by default from 5.26. On earlier releases, enable the
replacement by adding `QUERY_API_ENDPOINTS` to `server.http_enabled_modules`.

Programmatic branching on error-message text is deprecated from 2025.04
because messages can change. Parse GQLSTATUS error codes instead.

## Change Data Capture

`db.cdc.current()` returns `txCommitTime` alongside the transaction identifier
(2026.06.0), allowing a CDC client to obtain its most recent transaction's
commit time.

The beta procedures `cdc.current()`, `cdc.earliest()`, and `cdc.query()` are
deprecated. Use `db.cdc.current()`, `db.cdc.earliest()`, and `db.cdc.query()`.

## GenAI plugin

The GenAI plugin adds `GENAI_AZURE_OPENAI_BASE_URL` to change the base URL used
by `ai.text` calls (2026.04.0). It also adds:

- `ai.text.chunkByTokenLimit` to split input within a token limit; and
- `ai.text.countToken` to estimate an input's token count.

`ai.file.embedBatch` reads text from a local or remote file and generates
embeddings (2026.05.0). It can split input into chunks and returns one row per
chunk with its index, content, and embedding vector.

## Java and notification integrations

The server-side Notification API and Result Core API
`getNotifications()` method are deprecated from 5.26. Java integrations should
stop using these notification entry points.

Cypher 25 administrative commands using `WAIT` deliver cluster state as
notifications rather than result rows (2025.06). Consumers must use the
appropriate supported notification channel while it remains available.

## Fleet security logs

Security logs from self-managed Enterprise deployments can be collected for
Aura Console Security Log Analyzer (2026.04.0). The deployment must be
registered with Fleet Manager and log collection requires explicit opt-in.

## Helm and object storage

Non-TLS/SSL MinIO endpoints in the `neo4j/neo4j-admin` Helm charts are
deprecated. Configure the replacement `s3Endpoint`.

Cloud seed credentials must use each provider's built-in mechanism; Cypher 25
removes the `seedCredentials` database option (2025.06).
