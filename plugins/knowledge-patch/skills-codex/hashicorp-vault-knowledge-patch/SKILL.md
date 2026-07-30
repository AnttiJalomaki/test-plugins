---
name: hashicorp-vault-knowledge-patch
description: HashiCorp Vault
version: 2.0
license: MIT
metadata:
  author: Nevaberry
---


# HashiCorp Vault Knowledge Patch

Use this skill when designing, upgrading, configuring, or troubleshooting Vault deployments whose behavior depends on recent server, CLI, API, plugin, or Enterprise changes.

Prefer the deployed edition, exact patch release, configuration, API responses, and observed behavior over general guidance. Many capabilities here are Enterprise-only, beta, experimental, or gated by a feature or environment flag.

## Reference index

| Reference | Topics |
| --- | --- |
| [operations-and-upgrades.md](references/operations-and-upgrades.md) | Upgrade blockers, integrated storage, Raft, seals, containers, listeners, configuration, and known issues |
| [authentication-and-policy.md](references/authentication-and-policy.md) | Auth methods, identity, ACLs, MFA, OAuth, SCIM, namespaces, and request authorization |
| [secrets-and-rotation.md](references/secrets-and-rotation.md) | Secrets engines, static and root rotation, Secret Sync, leases, cloud integrations, and recovery |
| [pki-and-cryptography.md](references/pki-and-cryptography.md) | PKI, ACME, SCEP, Transit, KMIP, managed keys, FIPS, and cryptographic constraints |
| [audit-events-and-telemetry.md](references/audit-events-and-telemetry.md) | Audit records, events, activity, utilization, billing, reporting, metrics, and rotation evidence |
| [plugins-ui-and-clients.md](references/plugins-ui-and-clients.md) | Plugin lifecycle, SDK and client changes, GUI routes and workflows, Terraform, and operators |

## Start with upgrade blockers

Before changing a Vault release, edition, image, or plugin:

1. Identify the exact current and target patch releases.
2. Inventory integrated storage, seals, auth mounts, secrets engines, external plugins, policies, agents, and UI-dependent runbooks.
3. Check the relevant known-issue and fixed-version notes in [operations-and-upgrades.md](references/operations-and-upgrades.md).
4. Confirm every Enterprise-only capability against the installed license and enabled feature flags.
5. Exercise recovery, unseal, login, rotation, renewal, and plugin-registration paths in a representative environment.
6. Retain rollback artifacts and do not assume that a schema or behavior change is reversible.

### Configuration changes that can stop startup

- Integrated storage requires an explicit `disable_mlock = true` or `disable_mlock = false`; omission prevents startup.
- Integrated storage rejects `performance_multiplier` values less than or equal to zero.
- Duplicate attributes in server HCL and policy definitions are errors. The temporary `VAULT_ALLOW_PENDING_REMOVAL_DUPLICATE_HCL_ATTRIBUTES` escape hatch only downgrades them to warnings.
- Enterprise IBM PAO licenses require a `license_entitlement` stanza.
- Enterprise Common Criteria mode restricts listener TLS suites and tightens PKI validation.

### Authentication and policy breaks

- Azure auth requires a bound group or service-principal ID.
- Stored `auth/azure/config` values take precedence over `AZURE_*` environment variables.
- Empty LDAP passwords are always rejected; remove reliance on `deny_null_bind`.
- Exact-match list comparison for policy `allowed_parameters` and `denied_parameters` is retired; use per-element matching.
- Identity entity merges require `sudo`.
- Rekey cancellation requires the operation nonce.

### Secrets-engine retirements and migrations

- Snowflake database password authentication is retired; migrate to key-pair authentication.
- The Active Directory secrets plugin is retired; migrate before upgrading.
- Built-in Vault Agent API proxy support is deprecated; move proxy workloads to Vault Proxy.
- The PKI `allow_token_displayname` role field is deprecated; replace it with explicit name constraints.
- The AWS `security_token` response field is deprecated; consume `session_token`.
- Azure secrets `password_policy` is unusable and deprecated because Microsoft Graph generates passwords.
- Centrify auth is no longer officially supported.

### Container and plugin hazards

- Containers run as the `vault` user by default in newer patch images.
- Current images cannot call `mlock()` through a built-in `cap_ipc_lock`; set `disable_mlock = true` and prevent swapping at the host or runtime level.
- External plugin registration requires an extracted artifact in the plugin directory.
- Several specific Enterprise patch releases cannot verify plugins signed with the renewed signing key; use the fixed patch releases listed in the operations reference.

## High-value feature quick reference

### Automated credential rotation

Rotation Manager supports schedules and TTL/period cadences, interpreted in UTC, across supported cloud, database, and directory integrations. Check these details before enabling it:

- Manual LDAP static-role rotation does not reset the automated TTL.
- Rotation retries can be bounded and exhausted entries can become orphaned.
- Azure static rotations need spacing to avoid propagation races.
- Database imports can skip the first automatic static-role rotation.
- Logs and events provide evidence of rotation successes and failures.

See [secrets-and-rotation.md](references/secrets-and-rotation.md) for target coverage and migration behavior.

### Snapshot recovery

Enterprise integrated-storage recovery can expose supported snapshot data without restoring the whole cluster. Use `X-Vault-Recover-Snapshot-Id` instead of the deprecated query parameter. Recovery accepts `RECOVER`, `POST`, or `PUT`; `vault recover -from` can restore to a different live path.

Automatic snapshot loading is controlled by `autoload_enabled`. Separate snapshot-management permissions from recovery permissions. A stuck loaded snapshot can be force-unloaded with the CLI or snapshot-load API.

### Workload identity and authorization

- SPIFFE auth accepts JWT- or X.509-based SPIFFE IDs.
- Authenticated workloads can request JWT-SVIDs from Vault.
- Enterprise Agent Registry and OAuth resource-server support can authorize registered agents with OAuth 2.0 JWTs without a Vault token.
- SCIM 2.0 provisioning manages entities, aliases, and groups.
- Secret Sync can use workload identity federation for AWS, Azure, and GCP destinations.

### Transit and key management

- Transit supports envelope-encryption workflows so applications can process bulk data locally while Vault protects data-encryption keys.
- Recent algorithms include experimental ML-DSA and SLH-DSA, RSA PKCS#1 v1.5 encryption, and Enterprise Ed25519ph, Ed25519ctx, AES-CBC, and 192-bit AES CMAC.
- Derived datakey endpoints accept `context`, and rewrap supports managed keys.
- Managed-key usage values are strings such as `encrypt`, `sign`, and `mac`, not numeric IDs.
- Random-byte requests can be larger but consume correspondingly more memory.

Use [pki-and-cryptography.md](references/pki-and-cryptography.md) for constraints and protocol-specific behavior.

### PKI issuance and validation

Validate issuer extended-key usages, name constraints, issuer-name extensions, path length, chain usability, requested TTL, and role subject constraints together. Common Criteria mode adds full-chain and validation-time checks.

For ACME, constrain HTTP-01 and TLS-ALPN-01 targets with permitted and excluded IP ranges. Enterprise External CA and Vault Agent can run public-CA ACME workflows, and Agent templates re-render after issue or renewal.

### Events, audit, and billing

- Storage-changing events with `modified=true` carry `vault_index` for consistent follow-up reads.
- Response audit entries may carry `supplemental_audit_data`; PKI OCSP details remain HMACed by default.
- Activity query boundaries align to billing periods.
- Utilization bundles use `snapshot_records`; `decoded_snapshot` contains the readable data.
- Billing overview supports month ranges and configurable historical retention.

See [audit-events-and-telemetry.md](references/audit-events-and-telemetry.md) before changing parsers, dashboards, or billing exports.

## Request and listener compatibility

### Token header limit

Listeners cap `X-Vault-Token` and `Authorization: Bearer` content at 8 KB by default. If an existing integration legitimately needs larger tokens, assess the memory and exposure implications before disabling the limit:

```hcl
max_token_header_size = -1
```

### Privileged unauthenticated endpoints

Generate-root, DR operation-token generation, and rekey endpoints authenticate callers by default. Restore legacy unauthenticated behavior only after a deliberate threat review:

```hcl
enable_unauthenticated_access = ["generate-root", "generate-operation-token", "rekey"]
```

### Canonical paths

Clients should emit canonical paths. Requests containing `/./`, `/../`, or `//` may redirect to a cleaned path. Treat trailing-slash LIST authorization separately because more-specific denies now take precedence.

### JSON limits and forwarding

Listeners can bound JSON nesting, string length, object entries, and array elements. Rate-limit quotas run before these checks. When proxying to plugins, Vault strips Vault tokens from `Authorization` unless that header is explicitly allowed through; forwarded client IPs must parse as IPv4 or IPv6.

## Edition and maturity checks

Do not infer availability from an API name alone:

- Verify Community versus Enterprise edition.
- Distinguish generally available, beta, and experimental functionality.
- Verify whether a feature depends on managed keys, PKCS#11, seal entropy, a licensed add-on, or a specific plugin type.
- Confirm whether an auth or secrets plugin is built in, external, official-download eligible, or pinned.
- For GUI workflows, keep an equivalent CLI or API procedure where known UI issues can block work.

## Change-review checklist

For every proposed configuration or code change:

- Resolve the mount path and plugin type; aliases are not always equivalent.
- Preserve nonce, snapshot ID, lease, and rotation-schedule state where required.
- Check field renames, changed response types, deprecated parameters, and removed endpoints.
- Confirm UTC assumptions and cloud-provider propagation delays.
- Check whether a write is now persistent and therefore needs an explicit zero value to clear a field.
- Validate ACL, `sudo`, EGP, Sentinel, namespace, and role-based quota effects.
- Re-test proxies, canonical paths, forwarded headers, and certificate metadata.
- Bound memory-sensitive operations such as CRLs, JSON parsing, random bytes, and recovery.
- Capture audit, event, and telemetry evidence for the changed path.
- Consult every topic reference that intersects the deployment before rollout.

## Troubleshooting order

1. Reproduce with the exact API path, method, headers, namespace, and mount type.
2. Compare the installed patch release with the issue-specific fixed versions.
3. Inspect server logs, audit entries, health and seal status, Raft state, and plugin registration.
4. Distinguish authorization failure from routing, canonicalization, quota, JSON-limit, or header-limit failure.
5. Check provider-side propagation and external-secret state before retrying destructive rotation or sync operations.
6. Use the CLI or API when a documented GUI pagination, routing, or EGP interaction may be involved.
7. Escalate unresolved HSM, seal-wrap quorum, or recovery issues with preserved diagnostics and rollback state.
