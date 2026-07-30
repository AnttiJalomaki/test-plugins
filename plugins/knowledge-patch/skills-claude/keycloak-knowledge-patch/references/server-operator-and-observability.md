# Server, Operator, and Observability

## Environment configuration

### Preserve literal values

Values read from `KC_` environment variables undergo expression evaluation.
That includes resolving `${...}` and collapsing `$$` to `$`, which can alter a
secret without an obvious error.

Use the equivalent `KCRAW_` name to preserve dollar characters exactly.
Defining both forms for the same key is a startup error.

```bash
export KCRAW_DB_PASSWORD='my$$pa${vault}word'
```

### Map exact option keys

Environment-name normalization cannot round-trip every option name, notably
logging categories containing underscores. Pair an arbitrary `KC_` variable
with a same-suffix `KCKEY_` variable that supplies the exact option key.

```bash
export KC_MYKEY=debug
export KCKEY_MYKEY=log-level-package.class_name
```

## Optimized builds and provider artifacts

Every build option is stored in plaintext, even when it comes from the Java
KeyStore configuration source. Never place a secret in a build option.

With `start --optimized`, a repeated build option is ignored when it matches
the built value and rejected when it differs. Run another build to change it.

```bash
bin/kc.sh build --db=postgres
bin/kc.sh start --optimized
```

Container tooling can change provider JAR modification times between build and
runtime, making startup report that a provider changed. Assign deterministic
timestamps before `kc.sh build`.

```dockerfile
ADD --chown=keycloak:keycloak --chmod=644 some-jar.jar /opt/keycloak/providers/
RUN touch -m --date=@1743465600 /opt/keycloak/providers/*
RUN /opt/keycloak/bin/kc.sh build
```

## Request queue, bootstrap, and shutdown

The HTTP request queue is unlimited by default. Set
`http-max-queued-requests` to bound waiting requests; excess requests receive
an immediate HTTP 503.

```bash
bin/kc.sh start --http-max-queued-requests=1000
```

When health endpoints are enabled, asynchronous bootstrap can open HTTP(S) and
management listeners before initialization completes. Startup and liveness may
be UP while readiness remains DOWN. Route user traffic only after
`/health/ready`, or delay listener opening:

```bash
bin/kc.sh start --server-async-bootstrap=false
```

The 26.7.0 graceful-shutdown timeout defaults to ten seconds rather than one
second. Clustered nodes also wait for cache rebalance. Apply rolling changes
one node at a time; use `shutdown-timeout=1s` only when the former behavior is
deliberately required.

## Health and metrics

Additional datasources can be individually excluded from health checks
(26.7.0). Use this for optional datasources whose outage must not mark the
entire deployment unhealthy.

Keycloak 25 enables embedded-cache and HTTP server metrics by default. Health
and metrics are exposed on the separate management listener at port `9000`,
not the application ports.

`--legacy-observability-interface=true` temporarily restores the former
listener placement.

Use these settings to control histogram output:

- `cache-metrics-histograms-enabled`
- `http-metrics-histograms-enabled`
- `http-metrics-slos`

## Outbound HTTP response cap

From 25, responses consumed from brokers and other external services are capped
at 10 MB by default. Configure the byte limit with
`spi-connections-http-client-default-max-consumed-response-size`.

```bash
bin/kc.sh start \
  --spi-connections-http-client-default-max-consumed-response-size=1000000
```

## Runtime cache configuration

`cache`, `cache-stack`, and `cache-config-file` stop being build options in 25.
They are runtime-only. Remove them from image-build commands; otherwise the
server can silently fall back to its runtime cache defaults.

In 26, sessions are persistent by default. The standard cache configuration
limits each session cache to 10,000 entries with one owner. Apply comparable
bounds in custom cache XML.

The old two-minute session idle-time grace period is removed. Revoked access
tokens are persisted across embedded-cache restarts by default. Opt out with
`spi-single-use-object-infinispan-persist-revoked-tokens`.

Remove the old persistent-session batching options
`spi-user-sessions--infinispan--use-batches` and
`spi-user-sessions--infinispan--max-batch-size` in 26.7.0.

## Multi-cluster and external Infinispan

Preview multi-cluster v2 removes the external Infinispan cluster and fencing
infrastructure (26.7.0). Keycloak nodes use embedded caches, treat the
synchronously replicated database as the source of truth, and propagate
invalidations through a database-backed outbox. Enable this architecture with
the `stateless` feature.

For the older external-server topology, Keycloak 25 requires Infinispan 15 or
newer and supports it for multi-site deployments. In 26, multi-site mode
ignores distributed-cache and remote-store XML; configure `cache-remote-*`
options or equivalent custom-resource fields.

A single-site external store is rejected unless the temporary experimental
`cache-embedded-remote-store` feature is enabled. Prefer persistent sessions
for single-site restart survival.

## Datasources, transactions, and schema migration

Keycloak 25 makes `transaction-xa-enabled` default to false, enables
transaction recovery, and stores recovery logs under
`data/transaction-logs`.

From 26, a deployment with multiple datasources may have at most one non-XA
datasource. Enable XA for the default datasource with
`--transaction-xa-enabled=true`; configure each additional datasource with
`quarkus.datasource.<name>.jdbc.transactions=xa`.

Automatic database migration skips particular new indexes and prints SQL for
manual execution when the affected table has more than 300,000 rows:

| Version | Tables |
|---|---|
| 24 | `USER_ATTRIBUTE`, `FED_USER_ATTRIBUTE` |
| 25 | `RESOURCE_SERVER_PERM_TICKET` |
| 26 | `IDENTITY_PROVIDER` |

Capture and run the emitted statements after startup; do not assume automatic
migration created these indexes on large installations.

### PostgreSQL asynchronous commit

In 26.7.0, PostgreSQL transactions that touch only ephemeral session,
login-failure, or event tables use asynchronous commit. Logout stays
synchronous. Disable the optimization when required:

```bash
bin/kc.sh start --spi-connections-jpa--quarkus--async-commit=false
```

## Truststores and FIPS

Replace `spi-truststore-file-*` and truststore-related
`https-trust-store-*` settings with `conf/truststores` or
`truststore-paths`. Replace the old hostname-verification policy with
`tls-hostname-verifier`.

Because the truststore is always populated, direct WebAuthn attestation
requires the authenticator CA to be trusted. The default verifier changes from
`WILDCARD` to `DEFAULT` in 25. In 26, keystore and truststore type is inferred
from extensions such as `.p12`, `.jks`, and `.pem` unless overridden.

Generated system truststores sourced from `conf/truststores` or
`--truststore-paths` use BCFKS under strict FIPS in 26.7.0, allowing BCFIPS to
load them in approved mode. Default and non-strict FIPS deployments continue
to use PKCS12 where supported.

## Hostname and reverse proxy

Hostname v2 is the default from 25. `hostname` accepts a hostname or full URL,
while `hostname-admin` requires a full URL. Separate hostname path and port
options are gone. A full HTTPS URL selects HTTPS.

Dynamic backchannel resolution requires
`hostname-backchannel-dynamic=true` together with a full frontend URL.

Keycloak 26 removes hostname v1 and `proxy`. Replace edge or re-encrypt
configurations with one trusted `proxy-headers` format plus the appropriate
hostname and HTTP settings.

```bash
bin/kc.sh start \
  --hostname=https://sso.example.com:8543/auth \
  --proxy-headers=xforwarded \
  --http-enabled=true
```

## Operator installation and reconciliation

The Operator can be installed declaratively on vanilla Kubernetes with
kustomize (26.7.0), rather than applying individual manifests.

Preview cluster-wide mode lets one Operator reconcile `Keycloak` resources in
all namespaces. Use OLM `AllNamespaces`, or the `cluster-wide` kustomization
overlay for non-OLM installs.

From 24:

- Referenced Secrets and ConfigMaps are polled, not watched; propagation can
  take roughly one minute.
- Advanced property keys move from `operator.keycloak` to
  `kc.operator.keycloak`.
- Missing custom-resource resource settings default to a `1700MiB` memory
  request and a `2GiB` limit.

Version 26 adds default pod affinities and no longer implicitly supplies
`proxy=passthrough`.

The experimental Client Admin API v2 powers the `KeycloakOIDCClient` and
`KeycloakSAMLClient` custom resources; see
`admin-account-and-organizations.md`.

## Container memory

Container images from 24 use percentage-based heap sizing instead of fixed
`-Xms` and `-Xmx` values. Maximum heap defaults to 70% of available container
memory. Always set a container memory limit or the calculation can use the
host's total memory.

## Bootstrap administrator

Keycloak 26 deprecates `KEYCLOAK_ADMIN` and `KEYCLOAK_ADMIN_PASSWORD`. Use the
general bootstrap options or their replacement environment variables for
initial access and recovery.

```bash
export KC_BOOTSTRAP_ADMIN_USERNAME=admin
export KC_BOOTSTRAP_ADMIN_PASSWORD=change-me
```

## Generated secrets and AES keys

New client secrets generated in 26.7.0 are always 86 characters. Ensure secret
stores and integrations allow that length.

New `aes-generated` providers default to 256-bit keys. Existing providers do
not change. Rotate by adding a higher-priority provider with a 32-byte key and
leave it in place until sessions encrypted with older keys have expired.
