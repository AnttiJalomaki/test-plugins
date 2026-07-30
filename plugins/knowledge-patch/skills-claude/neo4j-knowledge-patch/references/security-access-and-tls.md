# Security, Access Control, and TLS

Use this reference when changing authentication, authorization, Fleet Manager
security-log collection, certificates, cipher policy, or administrative
credentials.

## Attribute-Based Access Control

ABAC applies to native users and native linked LDAP users as well as externally
authenticated SSO users (since 2026.06.0). Administrators can tag native DBMS
users and reference the tags in ABAC rules for dynamic role assignment. The
corresponding privilege is:

```text
DBMS USER METADATA MANAGEMENT
```

Audit who may manage metadata, because changing a tag can change effective role
assignment.

## Property-Based Access Control

A user-defined function cannot be part of a PBAC privilege. That combination
is unsupported and did not behave as its definition suggested. Replace such
rules with supported expressions.

## Authorization validation

Creating an auth rule containing an invalid time function now fails
immediately. It no longer defers the failure until the rule is evaluated during
authorization. Update deployment automation to surface creation-time errors.

Server-management procedures now require a specific privilege. Grant
`SERVER MANAGEMENT` to callers of:

```text
dbms.cluster.cordonServer()
dbms.cluster.setAutomaticallyEnableFreeServers()
dbms.cluster.uncordonServer()
```

Depending on an admin privilege for these calls is deprecated.

## OIDC flow and settings

`dbms.security.oidc.<provider>.auth_flow` supports PKCE and Implicit. PKCE is
the default. The Implicit flow is deprecated and will be removed, so migrate
providers and clients to PKCE.

The older `dbms.security.oidc.<provider>.auth_params` and
`dbms.security.oidc.<provider>.client_id` entry points are also deprecated.
Move configuration to the provider's supported current settings.

## Fleet Manager security logs

A self-managed Enterprise Edition deployment can send security logs to the
Aura console Security Log Analyzer (since 2026.04.0). The DBMS must be
registered with Fleet Manager, and collection requires explicit opt-in. Do not
assume registration alone enables log transfer.

## TLS hostname verification

The default for `dbms.ssl.policy.*.verify_hostname` changes from `false` to
`true`. After an upgrade, TLS policies verify peer hostnames unless existing
configuration explicitly pins another value. Ensure certificates contain the
names clients actually use; do not disable verification as a substitute for
correct identities.

## Post-quantum hybrid key exchange

With the OpenSSL provider 3.5 or later, TLS can use `X25519MLKEM768`. It
combines X25519 with ML-KEM-768 for hybrid post-quantum protection. Confirm
provider availability and peer interoperability before making it mandatory
(available in the 2026.05.0 line).

## Cipher-suite defaults

From 2025.10, four Java 21 CBC-based cipher suites are removed from Neo4j's
defaults because they are insecure:

```text
TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384
TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256
TLS_DHE_RSA_WITH_AES_256_CBC_SHA256
TLS_DHE_RSA_WITH_AES_128_CBC_SHA256
```

They remain available only when configured explicitly. Remove client
dependence on them instead of restoring them to the default policy.

## Legacy RSA private keys

Neo4j can still load a PKCS #1 key whose header is:

```text
-----BEGIN RSA PRIVATE KEY-----
```

That key form is deprecated and will be removed. Replace affected server keys
with a supported representation before removal.

## Helm object-storage endpoints

Non-TLS/SSL MinIO endpoints in the `neo4j/neo4j-admin` Helm charts are
deprecated. Configure the replacement `s3Endpoint` and use a secure endpoint.

## Security upgrade checks

1. Inventory users, authentication sources, OIDC flows, ABAC metadata, PBAC
   rules, and server-management grants.
2. Validate auth rules during deployment rather than waiting for a user access
   attempt.
3. Confirm the explicit opt-in and data boundary before enabling Fleet Manager
   security-log collection.
4. Test certificate names with hostname verification enabled.
5. Remove legacy CBC requirements, PKCS #1 keys, Implicit OIDC flows, and
   insecure object-storage endpoints.
