---
name: hashicorp-vault-knowledge-patch
description: HashiCorp Vault
version: 2.0
license: MIT
metadata:
  author: Nevaberry
---


# HashiCorp Vault Knowledge Patch

Use this skill when upgrading or operating Vault, configuring auth and secrets
engines, integrating plugins and agents, or consuming Vault APIs. Check the
breaking-change guidance first, then open the topic reference that matches the
task. Distinguish Community Edition behavior from features explicitly marked
Enterprise.

## Reference index

| Reference | Topics |
| --- | --- |
| [Migration and known issues](references/migration-and-known-issues.md) | Retirements, required configuration changes, compatibility fixes, unresolved issues, and workarounds |
| [Server, cluster, and storage](references/server-cluster-and-storage.md) | Raft membership and recovery, seals, listeners, request handling, physical storage, diagnostics, and namespaces |
| [Authentication, identity, and policy](references/auth-identity-and-policy.md) | Auth methods, identity deduplication, aliases, MFA, SPIFFE, ACLs, Sentinel, quotas, and SCIM |
| [Secrets, rotation, and synchronization](references/secrets-rotation-and-sync.md) | Cloud and database secrets, Rotation Manager, static roles, Secret Sync, leases, and Terraform integrations |
| [PKI, Transit, and managed keys](references/pki-transit-and-managed-keys.md) | Issuance constraints, ACME/SCEP, Transit algorithms, FIPS, KMIP, managed keys, and random bytes |
| [Plugins, agents, and delivery](references/plugins-agents-and-delivery.md) | Plugin registration, external plugins, containers, Agent, Proxy, Secrets Operator, and client SDK changes |
| [Audit, events, billing, and UI](references/audit-events-billing-and-ui.md) | Audit records, event subscriptions, metrics, activity and utilization APIs, reports, and GUI behavior |

## Check these upgrade blockers first

### Memory locking and integrated storage

- With integrated storage, set `disable_mlock` explicitly to `true` or `false`;
  omission prevents startup.
- Current container images cannot call `mlock()`. Set `disable_mlock = true` and
  prevent swapping at the host or container-runtime layer.
- Do not copy the short-lived 1.19.17 `IPC_LOCK` assumption forward:
  1.19.18 removed the image's built-in `cap_ipc_lock`.

### Authentication and policy behavior

- Azure auth requires a bound group or service-principal ID. Its stored
  `auth/azure/config` values take precedence over `AZURE_*` environment
  variables from 2.0 onward.
- Empty LDAP passwords are always rejected. Remove the deprecated
  `deny_null_bind` setting.
- Exact whole-list matching for policy `allowed_parameters` and
  `denied_parameters` is gone; write policies for per-element “contains all”
  matching.
- Duplicate attributes in server HCL and policy files are errors. Use
  `VAULT_ALLOW_PENDING_REMOVAL_DUPLICATE_HCL_ATTRIBUTES` only as a temporary
  migration aid.

### Retired and deprecated integrations

- Snowflake database password authentication is retired; migrate to key-pair
  authentication.
- The Active Directory secrets plugin is retired; move to a supported
  integration before upgrading.
- Vault Agent's built-in API proxy is deprecated; move proxy workloads to
  Vault Proxy.
- `/sys/internal/counters/tokens` now returns `403 unsupported path`; replace
  callers rather than retrying it.
- The Centrify auth plugin is no longer officially supported.

### Cluster and automation compatibility

- Removed Raft nodes with existing data cannot rejoin and no longer serve
  requests. Reprovision them instead of retrying `sys/storage/raft/join`.
- Rekey cancellation requires the operation nonce; retain it in automation.
- Non-canonical API paths may redirect after earlier patch releases rejected
  them. Generate clean paths and do not depend on redirect behavior.
- Listeners default `max_token_header_size` to 8 KB for `X-Vault-Token` and
  bearer tokens. Use `-1` only when an unlimited header is intentional.
- Integrated storage rejects `performance_multiplier <= 0`.

## Apply urgent compatibility fixes

### Enterprise plugin signatures

Vault Enterprise 1.19.17, 1.20.11, 1.21.6, and 2.0.1 cannot verify Enterprise
plugins released on or after April 21, 2026. Existing registrations still
work. Upgrade to 1.19.18, 1.20.12, 1.21.7, or 2.0.2 or later in the matching
release line before registering a newly signed plugin.

### Azure credential propagation

- For intermittent dynamic-role creation failures, use 1.19.19, 1.20.13,
  1.21.8, or 2.0.3 or later in the matching release line.
- Space Azure static-role `static-rotate` calls by several minutes. Rapid
  rotations can race propagation and leave the previous credential for manual
  cleanup.

### Enterprise 1.19 hazards

Review the detailed issue matrix before an Enterprise 1.19 upgrade. It records
duplicate HSM keys, Snowflake key-pair refresh failures, lost events with
multiple clients, local auth-mount configuration behavior, Rotation Manager
mount tracking, the 1.19.16 container `setfcap` failure, and the 1.19.18
seal-wrapped Raft quorum issue.

## Common operational patterns

### Recover secrets from a Raft snapshot

Enterprise recovery can read, list, and restore KV v1, cubbyhole, database
static-role, database credential, and SSH CA data from a loaded snapshot.

- Send the snapshot ID in `X-Vault-Recover-Snapshot-Id`; the
  `recover_snapshot_id` query parameter is deprecated.
- Use `RECOVER`, `POST`, or `PUT`. `vault recover -from` can restore to a
  different live path.
- Remove a stuck snapshot with
  `vault operator raft snapshot unload -force` or
  `DELETE sys/storage/raft/snapshot-load/{snapshot_id}?force=true`.
- `autoload_enabled` can load generated automated snapshots for recovery.
  Snapshot-management and recovery permissions are separate.

### Configure automated rotation

Rotation Manager schedules root rotations by UTC schedule or TTL/period and
reports the time remaining to the next run. It supports the documented cloud,
database, and directory integrations.

- An LDAP manual rotation in Enterprise 2.0+ does not reset the automatic TTL.
  Toggle `disable_automated_rotation` to `true` and back to `false` to
  calculate a new `next_vault_rotation`.
- LDAP self-managed static roles require a secrets mount enabled as `ldap`,
  not the `openldap` built-in alias, plus `self_managed=true`.
- Retry policies can cap attempts and orphan an entry after exhaustion.
- Database imports can skip an initial static-role rotation, and PostgreSQL
  `rotation_statements` may be multiline.

### Harden request processing

- Configure `max_json_depth`, `max_json_string_value_length`,
  `max_json_object_entry_count`, and `max_json_array_element_count` for JSON
  bodies. Rate-limit quotas run before these checks.
- Vault strips a Vault token from a forwarded plugin `Authorization` header
  unless that header is explicitly passed through.
- Treat `X-Forwarded-For` values as client IPs only when they parse as IPv4 or
  IPv6.
- Privileged generate-root, DR operation-token, and rekey endpoints
  authenticate by default. Restore legacy behavior only with an explicit
  `enable_unauthenticated_access` list.

### Operate Secret Sync safely

- `force_delete` defaults to false. Forcing deletion when associations cannot
  be unsynced leaves provider-side secrets orphaned.
- Removing the latest KV v2 version syncs the highest remaining active version
  instead of deleting the external value.
- IP and port allowlists constrain destinations; GCP destinations can use
  customer-managed encryption keys.
- Workload identity is supported for AWS, Azure, and GCP destinations.
  Deleting or disabling a source secrets-engine mount immediately unsyncs its
  external secrets.

## Common authentication patterns

### Certificate and proxy authentication

- `x_forwarded_for_client_cert_header` accepts RFC 9440 colon-wrapped Base64
  certificates.
- `enable_metadata_on_failures` can place client-certificate metadata in
  failed-login responses and audit records.
- Certificate roles can constrain `allowed_organizations`. Non-CA matching
  compares certificate equality, renewal requires the session certificate,
  and role-based quotas apply.

### Workload identity

- SPIFFE auth accepts JWT- or X.509-based SPIFFE IDs; authenticated workloads
  can also request JWT-SVIDs.
- Enterprise agent authorization lets Vault validate OAuth 2.0 JWTs for
  registered agent entities without a Vault token. Agent Registry support is a
  public beta for all customers by 2.0.3.
- `alias_metadata` can populate alias custom metadata for the supported auth
  methods.

## Common cryptography and PKI patterns

### Enforce issuer and leaf constraints

PKI issuance enforces issuer extended-key-usage, name constraints, issuer-name
extensions, signing-CA path length, and configured maximum TTL. Use
`leaf_not_after_behavior = "always_enforce_err"` to reject overlong TTLs for
leaf, CA, and ACME issuance.

- Cap revocation lists with `max_crl_entries`.
- Restrict ACME HTTP-01 and TLS-ALPN-01 targets with permitted and excluded IP
  ranges.
- Freshest CRL distribution points are supported at mount and issuer scope,
  and base CRLs carry the Freshest CRL extension.

### Select current algorithms deliberately

Transit supports experimental ML-DSA and SLH-DSA signatures, RSA PKCS#1 v1.5
encryption, and envelope encryption. Enterprise adds Ed25519ph/Ed25519ctx,
AES-CBC, AES-CMAC-192, derived-DEK context, and managed-key rewrap. FIPS builds
use a FIPS 140-3 module, while TLS can negotiate X25519MLKEM768.

For complete field names, edition boundaries, limits, UI changes, observability
schemas, and known-issue workarounds, use the indexed references rather than
inferring behavior from an older configuration.
