# Secrets, Rotation, and Synchronization

## Rotation Manager

- Rotation Manager rotates root credentials on a UTC schedule or by TTL/period.
  Enterprise integrations cover AWS auth and secrets, database secrets, GCP
  auth and secrets, Azure auth and secrets, and LDAP auth and secrets.
- Snowflake supports scheduled root rotation with key-pair credentials.
- Enterprise Rotation Manager exposes time remaining until the scheduled
  rotation.
- Server logs include success and failure details for root rotations and
  database or LDAP static-role rotations.
- Retry policies configure attempt limits; an entry can become orphaned after
  exhausting its attempts.
- 1.19.19 fixes routing for local mount entries under namespaces. Rotation
  Manager can still lose track after mount migration, so reconcile entries.

## Database and LDAP static roles

- Enterprise database imports can skip the initial automatic static-role
  rotation.
- PostgreSQL `rotation_statements` accept multiline statements.
- Enterprise LDAP gains self-managed static roles that rotate with their own
  password rather than requiring `bindpass`, plus schedules and retry
  policies.
- Existing LDAP static roles migrate from the plugin queue to Rotation Manager;
  inspect and manage progress through `static-migration`.
- Self-managed LDAP static roles require a mount enabled with type `ldap`, not
  the `openldap` built-in alias:

```shell
vault secrets enable -path=<mount_path> ldap
vault write <mount_path>/config self_managed=true
```

- A manual LDAP static-role rotation in Enterprise 2.0+ does not reset its
  automatic-rotation TTL. Toggle `disable_automated_rotation` to `true`, then
  `false`, to recalculate `next_vault_rotation`.
- LDAP auth root-password rotation supports a distinct rotation URL.
- Enterprise Active Directory root-password rotation has a `schema` field;
  `openldap` is the compatibility default.
- LDAP secrets supports IBM RACF static-role password phrases.
- Self-managed database static roles honor `escaping` or `disable_escaping`.

## Cloud secrets

- AWS secrets-engine writes persist fields, enabling partial updates. To clear
  an existing field, explicitly write its zero value.
- Enterprise AWS secrets supports cross-account management of static roles.
- AWS STS configuration has fallback endpoint and region fields; root
  configuration has `sts_region`.
- AWS AssumeRole and FederationToken responses use `session_token`;
  `security_token` is deprecated.
- Enterprise Azure secrets supports static roles. It adds role metadata,
  separates static-credential import, and lowers the minimum static-role TTL to
  30 days.
- Space Azure `static-rotate` calls by several minutes to avoid propagation
  races that retain old credentials.
- Azure dynamic-role creation propagation failures are fixed in 1.19.19,
  1.20.13, 1.21.8, and 2.0.3 or later in their respective lines.
- Azure `password_policy` is deprecated and ineffective because Microsoft
  Graph creates the password.
- Enterprise beta cloud import moves KV-compatible secrets from AWS, Azure, or
  GCP into Vault.

## Database engines and private connectivity

- MSSQL lease revocation requires only `VIEW ANY DEFINITION`, not `sysadmin`.
  Custom revocation statements execute as one batch rather than splitting at
  semicolons.
- Enterprise MSSQL EKM lets administrators choose the Transit key versions
  that wrap and unwrap SQL Server data-encryption keys.
- Database secrets supports Private Service Connect for GCP Cloud SQL MySQL
  and PostgreSQL, plus Private IP for MySQL.
- Snowflake password authentication is retired; use key-pair credentials.
- Enterprise Azure and database pinned-version overrides are described in the
  plugin reference.
- The Terraform Cloud secrets engine creates dynamic team tokens.

## KV metadata and Terraform

- KV v2 versions carry attribution metadata available from the CLI and API.
- The Enterprise Vault provider supports Terraform ephemeral resources and
  write-only attributes with KV and database secrets engines.

## Secret Sync

- GCP destinations support user-managed encryption keys.
- Destination configuration supports IP and port allowlists.
- `force_delete` defaults to false. It can delete a destination whose
  associations cannot be unsynced, but provider-side secrets remain orphaned.
- If the newest KV v2 version is removed, synchronization falls back to the
  highest active version instead of deleting the external secret.
- GitHub Enterprise Server destinations accept `enterprise_url`.
- Secret Sync supports workload identity federation and UI configuration for
  AWS, Azure, and GCP destinations.
- Deleting or disabling a secrets-engine mount immediately unsyncs its external
  secrets.

## Leases, local accounts, and protected delivery

- Enterprise `remove_irrevocable_lease_after` deletes irrevocable leases after
  the configured time past expiry. Nonzero values have a two-day minimum.
- The OS secrets engine rotates Linux local-account credentials.
- Vault Secrets Operator can map protected secrets into application pods as
  CSI-backed shared volumes without creating Kubernetes Secret objects.
