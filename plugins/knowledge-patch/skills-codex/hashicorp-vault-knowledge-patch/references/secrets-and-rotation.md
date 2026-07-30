# Secrets engines, sync, and rotation

Use this reference for secrets-engine configuration, credential lifecycle, Rotation Manager, Secret Sync, leases, snapshot recovery, and provider-specific compatibility.

## Rotation Manager fundamentals

- Rotation Manager schedules root credential rotation using a schedule or TTL/period. Schedule expressions are interpreted in UTC.
- Enterprise targets include AWS auth and secrets, database secrets, GCP auth and secrets, plus Azure and LDAP auth and secrets integrations.
- Snowflake root rotation supports key-pair credentials.
- Time remaining until an Enterprise scheduled rotation is exposed for operational visibility (since 1.21).
- Retry policies can cap attempts. Entries that exhaust their attempt allowance can be orphaned (since 2.0).
- Server logs provide success and failure details for automated root rotation and database or LDAP static-role rotation (since 1.21).

## Static-role scheduling

- Enterprise database imports can skip the initial automatic rotation of imported static roles.
- PostgreSQL `rotation_statements` support multiline statements.
- Self-managed database static roles honor their configured `escaping` or `disable_escaping` state (since 1.21).
- A manual LDAP static-role rotation in Enterprise 2.0+ does not reset the automated-rotation TTL. To begin a new cadence, set `disable_automated_rotation=true` on the role and then restore it to `false`; this recalculates `next_vault_rotation` (`upgrade-safety`).

## LDAP secrets engine

- The Enterprise LDAP secrets engine supports IBM RACF static-role password phrases.
- Enterprise 2.0 adds self-managed static roles that rotate with their own password instead of requiring `bindpass`. These roles support schedules and retry policies.
- Existing static roles migrate from the plugin queue into Rotation Manager. Track migration through the `static-migration` endpoint.
- Self-managed roles do not work when the mount uses the `openldap` built-in type alias. Enable the engine with type `ldap`, then turn on self-management (`upgrade-safety`):

```shell
vault secrets enable -path=<mount_path> ldap
vault write <mount_path>/config self_managed=true
```

- LDAP secrets emits rotation-success, rotation-failure, and other events (since 1.21).

## Active Directory retirement

The Active Directory secrets plugin is retired in the 1.19 line. Migrate workloads to a supported alternative before upgrading; do not confuse this plugin retirement with LDAP auth root-password rotation.

## AWS secrets

- AWS secrets-engine writes persist fields, so partial updates preserve omitted values. To clear a stored field, explicitly write its zero value.
- Enterprise supports cross-account management of AWS static roles.
- STS configuration accepts fallback endpoint and region values, while root configuration accepts `sts_region`.
- AssumeRole and FederationToken consumers should read `session_token`; `security_token` is deprecated (`upgrade-safety`).

## Azure secrets

- Enterprise Azure secrets adds static-role support in 1.21.
- In 2.0, Azure roles expose metadata, static-credential import is a separate operation, and the minimum TTL for static roles is 30 days.
- Allow several minutes between Enterprise `static-rotate` calls. Back-to-back rotation can race Azure propagation, fail to remove an old credential, and require manual provider-side cleanup (`upgrade-safety`).
- Dynamic-role creation can intermittently fail while a service principal propagates. Fixed patch floors are 1.19.19, 1.20.13, 1.21.8, and 2.0.3 for their respective lines.
- `password_policy` is deprecated and unusable because Microsoft Graph generates and returns passwords instead of accepting a requested password. Remove dependencies on Vault-generated Azure passwords (`upgrade-safety`).

## Database connectivity and revocation

- The database secrets engine supports Private Service Connect for GCP Cloud SQL MySQL and PostgreSQL, and Private IP for MySQL (since 1.21).
- MSSQL lease revocation requires `VIEW ANY DEFINITION`, not `sysadmin`.
- Custom MSSQL revocation statements run as one batch rather than being split at semicolons.

## Snowflake authentication

- Password authentication was deprecated in the 1.20 release line.
- It is retired in 1.21.x and can no longer be used. Migrate to key-pair credentials.
- Key-pair credential refresh has an unresolved issue in 1.19.x with an available workaround; validate the exact patch even after migrating.

## Terraform Cloud secrets

The Terraform Cloud secrets engine can issue dynamic team tokens (since 1.20).

## OS local accounts

The OS secrets engine can automatically rotate Linux local-account credentials (since 2.0).

## Secret Sync controls

- Enterprise GCP destinations support user-managed encryption keys.
- Destination configuration accepts IP and port allowlists.
- `force_delete` defaults to false. It can delete a destination when associations cannot be unsynced, but provider-side secrets remain orphaned; inventory and clean them separately.
- If the latest KV v2 version is removed, sync selects the highest remaining active version instead of deleting the external secret.
- GitHub destinations accept `enterprise_url` for self-hosted GitHub Enterprise Server (since 1.21).
- Workload identity federation is supported for AWS, Azure, and GCP destinations and can be configured in the UI (since 2.0).
- Deleting or disabling a secrets-engine mount immediately unsyncs its external secrets (since 2.0). Treat mount lifecycle changes as provider-impacting operations.
- Enterprise secret-deletion event subscriptions do not require a root token from 1.19.

## Cloud secret import

Enterprise beta import can bring key-value-compatible secrets from AWS, Azure, and GCP into Vault. Plan naming, ownership, and deletion behavior before treating import as a migration cutover.

## Lease behavior

- `vault lease renew --fail-if-not-fulfilled` exits with failure when Vault cannot satisfy the requested renewal, so shell command chaining can reliably stop.
- The default API client honors `Retry-After`; delays are rounded up to whole seconds.
- Enterprise `remove_irrevocable_lease_after` automatically removes irrevocable leases after the configured duration beyond expiry. Zero disables removal; a nonzero duration must be at least two days.

## KV v2 attribution

KV v2 versions carry attribution metadata that is visible through the CLI and API (since 1.21). Preserve this data when building inventory or change-history tooling.

## Integrated-storage recovery

Enterprise can load a Raft snapshot and read, list, or recover KV v1 and cubbyhole values. Later 1.20 patches extend recovery to database static roles and credentials and to the SSH plugin CA.

- Send the snapshot ID in `X-Vault-Recover-Snapshot-Id`; `recover_snapshot_id` is deprecated.
- Recovery accepts the `RECOVER` method as well as `POST` and `PUT`.
- `vault recover -from` restores an item to a different live path.
- Automated snapshot configuration accepts `autoload_enabled`; generated snapshots are automatically loaded when enabled.
- Snapshot-management and recovery permissions are separate, allowing recovery to be delegated without snapshot administration.
- Remove a stuck loaded snapshot with `vault operator raft snapshot unload -force` or `DELETE sys/storage/raft/snapshot-load/{snapshot_id}?force=true`.

## Rotation validation checklist

1. Resolve UTC schedules and whether manual action should preserve or reset cadence.
2. Check whether the role is managed, self-managed, imported, or still being migrated.
3. Space cloud-provider rotations to accommodate propagation.
4. Confirm retry limits and monitor for orphaned entries.
5. Verify provider-side deletion after force delete, mount disable, or failed rotation.
6. Capture logs, LDAP events, and KV version attribution as evidence.
