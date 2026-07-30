# Migration and Known Issues

Use this reference while planning an upgrade or diagnosing behavior that
changed across patch lines. Evolution within the source is resolved here as a
sequence, not treated as a contradiction. Batch attributions include
`1.19-changelog`, `1.19`, `1.20-changelog`, `1.20`, `1.21-changelog`, `1.21`,
`2.0-changelog`, `2.0`, and `upgrade-safety`.

## Required configuration and behavior changes

### Memory locking and containers

- Container execution changed during the `1.19-changelog` batch. Images run as
  the `vault` user by default from 1.19.16. The 1.19.17 image required runtime
  `IPC_LOCK`, but 1.19.18 removed built-in `cap_ipc_lock`; containers can no
  longer call `mlock()`. Configure `disable_mlock = true` and disable swapping
  in the runtime or host.
- With integrated storage (batch `1.20-changelog`), `disable_mlock` has no
  default. Set it explicitly to `true` or `false` or Vault refuses to start.
- The 1.19.16 image has a separate unresolved startup failure related to
  `setfcap`; use the published workaround when that exact image is unavoidable.

### HCL and policy migration

- Duplicate server-HCL and policy attributes were deprecated in 1.19. They are
  errors in the `1.21-changelog` behavior. The temporary
  `VAULT_ALLOW_PENDING_REMOVAL_DUPLICATE_HCL_ATTRIBUTES` switch downgrades them
  to warnings while configurations are cleaned up.
- `VAULT_NEW_PER_ELEMENT_MATCHING_ON_LIST` opted into per-element “contains
  all” checks for `allowed_parameters` and `denied_parameters` in 1.19.
  Exact-match list comparison is retired in the `1.21` line, so policies must
  use the per-element semantics.
- `resultant-acl` now merges segment-wildcard (`+`) paths with prefix rules in
  `glob_paths`; consumers should expect the complete combined view.
- Enterprise soft-mandatory Sentinel policy overrides now honor the request's
  override flag and may allow a request that the policy initially denied.

### Auth and secrets migrations

- Azure auth requires a bound group or service-principal ID from `1.20`.
- From 2.0, stored `auth/azure/config` values override `AZURE_*` environment
  variables. Move intended overrides into the stored configuration.
- Empty LDAP passwords are always denied. The `deny_null_bind` setting is
  deprecated and no longer changes behavior.
- The Active Directory secrets-engine plugin is retired in `1.19`; migrate
  before upgrading.
- Snowflake database password authentication is deprecated in `1.20` and
  retired in `1.21`; use key-pair authentication.
- The Centrify auth plugin is no longer officially supported.
- AWS AssumeRole and FederationToken consumers should read `session_token`;
  `security_token` is deprecated.
- Azure secrets `password_policy` is deprecated and unusable because Microsoft
  Graph generates the password; remove dependencies on Vault-side password
  generation.

### Endpoint, UI, and client migrations

- `/sys/internal/counters/tokens` is deprecated and returns
  `403 unsupported path` in the `1.20-changelog` behavior.
- Secrets-engine UI routes move from `/secrets` to `/secrets-engines` in
  `2.0-changelog`; the list view also drops bulk deletion.
- Vault Agent's built-in API proxy is deprecated and pending removal. Migrate
  proxy behavior to Vault Proxy.
- PKI role `allow_token_displayname` is deprecated and targeted for removal in
  April 2027. Replace it with `allowed_domains`, `allow_bare_domains`,
  `allow_subdomains`, or `allow_glob_domains`.
- API plugin clients should move from deprecated `RegisterPlugin` and
  `RegisterPluginWithContext` to detailed registration variants that return
  both the registration response and an error.
- Manual utilization bundles replace `snapshots` with `snapshot_records`;
  `decoded_snapshot` contains the former readable snapshot data.
- `GET sys/managed-keys/:type/:name` returns usage names (`encrypt`, `decrypt`,
  `sign`, `verify`, `wrap`, `unwrap`, `generate_random`, `mac`) instead of
  numeric IDs. Update strongly typed decoders.

## Cluster and request compatibility

- Removed Raft nodes with existing data are rejected by
  `sys/storage/raft/join`; they stop serving, shut down, and seal. Reprovision
  rather than repeatedly joining the old data directory.
- Starting in 1.19.6, rekey cancellation requires its nonce. Persist the nonce
  in operational automation.
- 1.19.16 began rejecting non-canonical paths; 1.19.19 redirects paths
  containing `/./`, `/../`, or `//` to their cleaned form. Always send
  canonical paths. A mount tuneable can trim POST trailing slashes, and
  trailing-slash LIST requests now apply more-specific deny rules.
- In `2.0-changelog`, listeners cap `X-Vault-Token` and
  `Authorization: Bearer` values with `max_token_header_size`, default 8 KB.
  Set `max_token_header_size = -1` only to opt out deliberately.
- Integrated storage rejects `performance_multiplier` values at or below zero.

## Enterprise plugin compatibility

Vault Enterprise 1.19.17, 1.20.11, 1.21.6, and 2.0.1 cannot register
Enterprise plugins released on or after April 21, 2026 because they cannot
verify the renewed signing key. Existing registrations are unaffected. Upgrade
to 1.19.18, 1.20.12, 1.21.7, or 2.0.2 or later within the corresponding release
line.

## Azure rotation compatibility

- Static-role rotations performed too close together in 1.21 and 2.0 can race
  Azure propagation, fail to remove the old credential, and require manual
  cleanup. Wait several minutes between `static-rotate` calls.
- Dynamic-role creation can fail while a new service principal propagates.
  Upgrade to 1.19.19, 1.20.13, 1.21.8, or 2.0.3 or later in the matching line.

## Enterprise 1.19 issue matrix

Vault Enterprise 1.19 is the current LTS line in its `1.19` batch; 1.16.x moved
to long-term support. Account for these unresolved or partially fixed hazards:

| Area | Behavior | Mitigation |
| --- | --- | --- |
| HSM | Duplicate unseal or seal-wrap HSM keys | Apply the release-note workaround |
| Snowflake | Key-pair credential refresh can fail | Apply the available workaround |
| Local auth mounts | Writes to local LDAP, AWS, GCP, or Azure auth mounts may ignore `local` | No workaround is listed |
| Events | Multiple connected event clients can miss events | Apply the available workaround |
| Rotation Manager | 1.19.19 fixes routing of local namespace mounts, but mount migration can still lose tracking | Reconcile entries after migration |
| Container | 1.19.16 may fail startup because of `setfcap` | Apply the available workaround |
| Raft | 1.19.18 seal wrapping can cause quorum failures | No workaround is listed |

## Open GUI issues

- In Enterprise 2.0, an Endpoint Governing Policy can deny a root token's child
  namespace GUI access when the GUI calls `sys/internal/ui/mounts`. Use CLI or
  API access, or explicitly permit that endpoint in the EGP.
- In 1.21 and 2.0, changing **Items per page** away from page 1 of Secrets
  Engines can render an empty or incomplete list. Return to page 1 first, or
  refresh and retry there.
