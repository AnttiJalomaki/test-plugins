# Security, Cluster Operations, and Storage

Use this reference for credentials, realms, TLS, entitlement behavior, cluster
allocation, repositories, snapshots, connectors, and operational APIs.

## Contents

- [Credentials and identity](#credentials-and-secure-settings)
- [TLS and entitlements](#tls-and-file-reload)
- [Cluster operations](#cluster-resolution-and-index-metadata)
- [Snapshots and repositories](#snapshot-and-archive-behavior)
- [Connectors and synonyms](#connector-apis)
- [Operational API changes](#operational-api-changes)

## Credentials and secure settings

- API-key credential hashes may use `SSHA-256` as of 9.0.0.
- Secure settings are no longer accepted in YAML configuration in 9.0.0; put
  them in Elasticsearch's secure-settings mechanism.
- Secure-settings reload responses include the names of reloaded settings and
  the keystore modification time in 9.3.0.
- API keys can be cloned through a dedicated endpoint in 9.4.0.
- Cross-cluster API keys gain signing and trust configuration in 9.2.0.
- Cross-cluster API keys carry and validate certificate identities in 9.3.0.
- The `read` index privilege consistently authorizes cross-cluster search in
  9.4.0 regardless of `ccs_minimize_roundtrips`.
- Service-account-token APIs are available in Serverless as of 9.4.0.

Keep paired credentials atomic: configuring an LDAP or Active Directory bind
DN without a bind password prevents startup.

## Realms, identity, and extensions

- `SecurityExtensions` can supply a custom `ServiceAccountTokenStore` in
  9.1.0.
- The SAML identity provider supports custom attributes in 9.1.0.
- JWT access tokens may use an `at+jwt` `typ` header in 9.1.0.
- A Microsoft Graph delegated-authorization realm plugin arrives in 9.1.0.
- SAML private attributes become configurable in 9.2.0.
- URL-based SAML metadata resolution gains configurable HTTP connect and read
  timeouts in 9.2.0.
- JWT realms can periodically reload PKC JWK sets in 9.3.0.
- Successful SAML responses include in-response-to metadata in 9.3.0.
- The `reporting_user` role changed in 9.0.6 and 9.1.3 to derive authorization
  from reserved Kibana privileges. Revisit custom assumptions about its old
  privilege composition.

## TLS and file reload

- TLS configuration reload watches SSL files rather than their containing
  directories in 9.1.0.
- General configuration reload detects Kubernetes CSI-style `..data` symlink
  switches in 9.1.0.
- Transport TLS handshake timeout is configurable in 9.2.0.
- TLS extensions can implement `SslProfileExtension` and receive reload
  notifications through an `SslProfile` listener in 9.2.0.
- Concurrent TLS handshakes can be limited in 9.3.0.
- On JDK 24, `TLS_RSA` ciphers are unsupported and TLSv1.1 is absent from the
  default protocol list.
- The `cloud-ess-fips` package defaults to FIPS 140-3 in 9.4.0.

## Entitlements and platform baseline

- Elasticsearch Entitlements permanently replaces the Java SecurityManager in
  9.0.0 because Java 24 disables the SecurityManager.
- Elasticsearch 9.0.0 bundles JDK 24, uses Lucene 10.1.0, and moves the default
  Docker image from Ubuntu to UBI minimal. Startup ignores `_JAVA_OPTIONS`.
- Elasticsearch 9.4.0 upgrades to Apache Lucene 10.4.
- Elasticsearch 9.0.0 entitlement paths are case-sensitive on Windows even
  when the filesystem is not. Match exact filesystem casing in settings,
  environment values, secure settings, and command-line paths.
- The 9.0.0 `x-pack-core` entitlement policy blocks Active Directory
  connectivity; use the compatibility-reference workaround or a fixed
  release.

## Cluster resolution and index metadata

- `_resolve/cluster` can query cluster information without an index expression
  and accepts a caller-configurable timeout in 9.0.0.
- `_resolve/index` can filter by index mode and returns the mode in 9.2.0.
- The Get Data Stream API reports index mode in 9.1.0.
- Cat APIs add a circuit-breakers endpoint in 9.3.0.
- Shard-capacity health thresholds become configurable in 9.3.0.

## Allocation and shutdown

- Shard-balancing weights can be configured independently per data tier in
  9.1.0.
- `replica_unassigned_buffer_time` defaults to five seconds rather than three
  seconds in 9.0.0.
- The single-data-node disk-watermark setting is removed.
- `/_cluster/reroute` responses no longer include cluster state.
- Persistent-task reassignment during node shutdown is opt-in in 9.4.0.
- Shutdown status reports shard snapshot pauses in 9.4.0.

## Snapshot and archive behavior

- Get Snapshots accepts a `state` query parameter in 9.1.0.
- Archive and searchable-snapshot indices accept N-2 versions in 9.0.0,
  including supported 7.x segment cases used as archives in 8.x or 9.x.
- Frozen indices are no longer readable, and the unfreeze endpoint is removed.
- ILM searchable snapshots can specify `replicate_for` in 9.0.0.

## S3 repositories

### SDK and metadata

- `repository-s3` supports IMDSv2 in 9.0.0.
- In 8.19.0, `repository-s3` moves from AWS SDK v1 to v2. SDK configuration
  and behavior differ, so exercise production-equivalent repository settings
  before upgrading.

### Integrity and transport controls

- S3 repositories use conditional writes in 9.2.0 to prevent accidental
  object overwrites and repository corruption, including on fully
  S3-compatible storage.
- Maximum S3 connection idle time is configurable in 9.2.0.
- S3 repository API-call timeout is configurable in 9.3.0.
- Before 9.3.0, repository analysis can falsely fail
  linearizable-register checks because of multipart-upload semantics. Use a
  one-node analysis with `?register_operation_count=1` or upgrade.

## EC2 discovery

The `discovery-ec2` plugin uses AWS SDK v2 and now:

- Requires IMDSv2.
- Ignores `discovery.ec2.protocol`; put the scheme in
  `discovery.ec2.endpoint`.
- Removes `aws.secretKey`.
- Removes `com.amazonaws.sdk.ec2MetadataServiceEndpointOverride`.
- Requires both `discovery.ec2.access_key` and `discovery.ec2.secret_key`, or
  neither.

## GCS repositories

`repository-gcs` operations using Application Default Credentials can fail in
9.2.8 and 9.3.3 because an entitlement exception escapes credential-path
discovery. Upgrade to 9.2.9 or 9.3.4; the compatibility reference contains the
temporary JVM-policy workaround.

## Connector APIs

- Connector delete APIs support soft and hard delete through a URL parameter
  in 9.0.0.
- Managed connector indices must use the required prefix.
- Connector APIs require `manage_connector` or `monitor_connector`.

## Synonyms

Synonyms PUT and delete APIs accept `refresh` in 9.1.0. It waits for changed
synonyms to become visible and reloads affected analyzers. Use it when a
workflow must not query with stale synonym state after an update.

## Operational API changes

- Remote reindex accepts a convenience API-key parameter in 9.3.0 and a
  blocklist setting in 9.4.0.
- Stateful cross-cluster operation disables `_delete_by_query` and
  `_update_by_query` in 9.3.0.
- Query logging spans `_search`, ES|QL, EQL, and SQL in 9.4.0, and the
  search-task watchdog can log hot threads for slow searches.
- Elasticsearch timeouts now return HTTP 429 instead of a 5xx status.
