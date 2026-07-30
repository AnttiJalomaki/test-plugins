# Deployment and Storage Migrations

## Truststore and TLS verification

Replace the 24-era `spi-truststore-file-*` and truststore-related
`https-trust-store-*` settings with `conf/truststores` or `truststore-paths`.
Replace the old hostname-verification policy with `tls-hostname-verifier`.
Because the truststore is always populated, direct WebAuthn attestation requires
the authenticator CA to be trusted.

Account for the verifier default changing from `WILDCARD` to `DEFAULT` in 25.
In 26, infer keystore and truststore type from `.p12`, `.jks`, or `.pem`
extensions unless an explicit type overrides inference.

## Hostname and reverse proxies

Hostname v2 is the default from 25. `hostname` accepts either a host or a full
URL, while `hostname-admin` requires a full URL. Remove separate path and port
options. Select HTTPS with a full HTTPS URL. Dynamic backchannel resolution
requires `hostname-backchannel-dynamic=true` and a full frontend URL.

Version 26 removes hostname v1 and `proxy`. Replace edge or re-encrypt setups
with one trusted `proxy-headers` format plus the appropriate hostname and HTTP
settings.

```bash
bin/kc.sh start \
  --hostname=https://sso.example.com:8543/auth \
  --proxy-headers=xforwarded \
  --http-enabled=true
```

## Password hashing and user-profile migration

The default password hash changes in 24 from PBKDF2-SHA256 to PBKDF2-SHA512 at
210,000 iterations. Passwords without an explicit realm policy rehash at login.
Version 25 makes Argon2 the non-FIPS default, causing another one-time rehash
and temporary database load, and changes the default garbage collector from
ParallelGC to G1GC.

## Preserving sessions across upgrades

To preserve online sessions from 24, upgrade to 25 with preview
`persistent-user-sessions` enabled on that first upgrade. Only sessions already
stored in remote Infinispan or embedded-cache JDBC persistence can migrate.
Enabling the feature later cannot safely merge persisted and non-persisted
sessions.

Version 26 changes cache marshalling from JBoss Marshalling to incompatible
Protostream and clears all caches. A direct upgrade that skipped the 25
persistence migration therefore loses sessions.

All sessions persist by default in 26. The standard cache file caps each
session cache at 10,000 entries with one owner; apply equivalent bounds in
custom cache XML. The former two-minute idle grace period is gone. Revoked
access tokens persist across embedded-cache restarts by default; opt out with
`spi-single-use-object-infinispan-persist-revoked-tokens`.

## Cache topology and runtime options

Move `cache`, `cache-stack`, and `cache-config-file` out of build commands in
25; they are runtime-only options, and putting them in image builds allows
runtime cache defaults to be selected silently.

External multi-site deployments require Infinispan 15 or newer from 25. In 26,
multi-site mode ignores distributed-cache and remote-store XML; use
`cache-remote-*` or corresponding custom-resource fields. Reject a single-site
external store unless the temporary experimental
`cache-embedded-remote-store` feature is enabled. Prefer persistent sessions
for single-site restart survival.

## Metrics and management endpoints

Version 25 enables embedded-cache and HTTP server metrics by default. Health
and metrics use the management listener on port `9000`, not the application
ports. `--legacy-observability-interface=true` temporarily restores the old
placement. Configure histograms with `cache-metrics-histograms-enabled`,
`http-metrics-histograms-enabled`, and `http-metrics-slos`.

## Outbound HTTP response limits

Responses consumed from identity brokers and other external services are
capped at 10 MB by default from 25. Change the byte limit using
`spi-connections-http-client-default-max-consumed-response-size`.

```bash
bin/kc.sh start --spi-connections-http-client-default-max-consumed-response-size=1000000
```

## Transactions and additional datasources

`transaction-xa-enabled` defaults to false from 25. Transaction recovery is
enabled and writes logs below `data/transaction-logs`. From 26, deployments
with multiple datasources may contain at most one non-XA datasource. Enable XA
for the default datasource with `--transaction-xa-enabled=true`; configure each
additional datasource with
`quarkus.datasource.<name>.jdbc.transactions=xa`.

## Large-table indexes

Automatic migration skips selected new indexes when the affected table already
has more than 300,000 rows and prints SQL for manual execution. This applies to
`USER_ATTRIBUTE` and `FED_USER_ATTRIBUTE` in 24,
`RESOURCE_SERVER_PERM_TICKET` in 25, and `IDENTITY_PROVIDER` in 26. Capture and
run the emitted statements after startup; do not assume migration created them.

## Bootstrap administration

Version 26 deprecates `KEYCLOAK_ADMIN` and `KEYCLOAK_ADMIN_PASSWORD`. Use
general bootstrap options or the replacement environment variables for initial
access and recovery.

```bash
export KC_BOOTSTRAP_ADMIN_USERNAME=admin
export KC_BOOTSTRAP_ADMIN_PASSWORD=change-me
```

## Operator and container resources

From 24, the Operator polls referenced Secrets and ConfigMaps rather than
watching them, so changes can take about one minute. Rename advanced property
keys from `operator.keycloak` to `kc.operator.keycloak`. When custom-resource
resource settings are absent, expect a `1700MiB` memory request and `2GiB`
limit. Version 26 adds default pod affinities and no longer implicitly supplies
`proxy=passthrough`.

Keycloak 24 container images replace fixed `-Xms` and `-Xmx` with percentage
sizing and default maximum heap to 70% of available container memory. Always
set a container memory limit so heap sizing does not use the host total.

## Removed runtime components

Version 25 no longer bundles the Oracle JDBC driver and removes the legacy
LinkedIn OAuth provider. Install a compatible Oracle driver and use the
remaining LinkedIn OIDC provider. Version 26 removes the GELF handler, adapter
and miscellaneous BOMs, `keycloak-test-helper`, and the JEE admin client; the
Jakarta admin client remains.

## LDAP pooling and binary attributes

Realm-level LDAP pool settings are ignored in 26 because connection pooling is
JVM-wide; migrate them to the documented system properties. In 26.7, existing
binary user-attribute mappers migrate to `base64`, while new mappers default to
`auto` and may explicitly choose `base64` or `uuid`. Existing group mappers
retain base64 behavior, and new group mappers enable UUID decoding.
