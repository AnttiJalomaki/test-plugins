# Operations and upgrades

Use this reference for cluster lifecycle, storage, seals, containers, listener limits, configuration compatibility, and release-specific hazards. Guidance is grouped by operator task rather than release chronology.

Extraction attribution for version-sensitive notes: `1.19-changelog`, `1.19`, `1.20-changelog`, `1.20`, `1.21-changelog`, `1.21`, `2.0-changelog`, and `2.0`. Additional cross-release guidance is attributed to `upgrade-safety`.

## Cluster membership and health

- Health responses now distinguish a removed node and an unhealthy HA standby. The default failure codes are `530` for removed and `474` for HA-unhealthy; override them with `removedcode` and `haunhealthycode`.
- `sys/seal-status` and `vault status` expose `removed_from_cluster`; seal status also exposes `migration_done_at_epoch` in later patches.
- A removed node with existing Raft data cannot rejoin through `sys/storage/raft/join`. Removed nodes stop serving requests and are shut down and sealed. Reprovision or clean the intended node state instead of repeatedly joining it.
- SIGHUP reloads additional Raft settings. `/sys/config/state/sanitized` exposes the sanitized active settings, which is useful for confirming a reload.
- Raft auto-join can be forced to IPv4 on dual-stack networks.
- Integrated storage rejects `performance_multiplier <= 0` (since 2.0).

## Memory locking and containers

- With integrated storage, `disable_mlock` has no default; set it explicitly to `true` or `false`, or Vault refuses to start (since 1.20).
- Container images run as the `vault` user by default from 1.19.16. The 1.19.17 image required runtime `IPC_LOCK`, but 1.19.18 removed the built-in `cap_ipc_lock`.
- Current containers therefore cannot call `mlock()`. Configure `disable_mlock = true` and disable swapping at the container runtime or host.
- The 1.19.16 Docker image has a `setfcap` startup failure with a documented workaround; account for this if that exact image is still deployed.
- Images are distributed as compressed OCI image layouts. UBI variants use UBI 10 minimal.

## Paths, parsing, and listener limits

- Vault 1.19.16 started rejecting non-canonical paths. From 1.19.19, paths containing `/./`, `/../`, or `//` redirect to their cleaned form.
- A mount tuneable can trim trailing slashes on `POST`. Trailing-slash `LIST` requests honor more-specific denies instead of falling through to broader allows.
- HTTP handling can bound request data with `max_json_depth`, `max_json_string_value_length`, `max_json_object_entry_count`, and `max_json_array_element_count`. Quotas are evaluated before JSON limits.
- Listeners cap `X-Vault-Token` and bearer-token header content with `max_token_header_size`, default 8 KB. A value of `-1` disables the limit (since 2.0).
- Client IP values obtained from `X-Forwarded-For` must parse as valid IPv4 or IPv6 addresses.
- Agent, Proxy, server, and other displayed configuration canonicalizes IPv6 values using RFC 5952.

## HCL migration

- Duplicate attributes in HCL server configuration and policy definitions were deprecated in the 1.19 line.
- They are errors from 1.21. `VAULT_ALLOW_PENDING_REMOVAL_DUPLICATE_HCL_ATTRIBUTES` temporarily restores warning behavior to aid migration; remove duplicates rather than treating the flag as permanent.

## Rekey, seal, and HSM safety

- Rekey cancellation requires the operation nonce from 1.19.6. Automation must retain the nonce and submit it when cancelling.
- Seal HA will not persist the barrier keyring unless every seal is healthy. Later 1.19 patches allow new nodes to join Seal-HA clusters.
- `detect_deadlocks` accepts `sealwrap`. When seal wrap is active, AppRole secrets are seal-wrapped.
- Selected sensitive seal-wrap and managed-key configuration values may be read from environment variables or files.
- Enterprise 1.19 has an unresolved duplicate unseal or seal-wrap HSM-key issue; use the release-note workaround.
- Enterprise 1.19.18 has an unresolved seal-wrap issue that can cause Raft quorum failure and has no listed workaround. Do not roll onto that patch without an explicit risk decision.

## Storage backends

- The DynamoDB storage backend can modify its table to use on-demand, per-request billing.
- The PostgreSQL physical backend can authenticate with AWS IAM, Azure MSI, or GCP IAM identities (since 1.20).
- The MySQL physical backend can obtain credentials from `VAULT_MYSQL_USERNAME` and `VAULT_MYSQL_PASSWORD` (since 1.20).

## Diagnostics and reload visibility

- Start the server with `pprof-dump-dir` to write startup profiles.
- `enable_post_unseal_trace` and `post_unseal_trace_directory` capture a Go trace after unseal.
- Use `sys/health`, `sys/seal-status`, `vault status`, sanitized configuration state, and the Raft APIs together when diagnosing membership or post-unseal failures.

## Upgrade compatibility issues

### Plugin signing key

Enterprise releases 1.19.17, 1.20.11, 1.21.6, and 2.0.1 cannot register Enterprise plugins released on or after April 21, 2026 because the renewed signing key fails verification. Existing registrations are unaffected. Upgrade, respectively, to 1.19.18, 1.20.12, 1.21.7, or 2.0.2 or later in the same release line (`upgrade-safety`).

### Azure propagation

- Dynamic-role credential creation can fail before a new service principal propagates. Use 1.19.19, 1.20.13, 1.21.8, or 2.0.3 or later in the applicable release line.
- Static-role rotations issued in rapid succession can fail to remove the previous credential. Wait several minutes between `static-rotate` calls and manually inspect for leftovers.

### Rotation and mount migration

- Vault 1.19.19 fixes incorrect routing of local mount entries under namespaces.
- Rotation Manager can still lose track of entries after a mount migration. Inventory scheduled entries before and after moving a mount.
- Configuration writes to local LDAP, AWS, GCP, or Azure auth mounts can ignore the mount's `local` flag in 1.19.x; no workaround is listed.

### Event delivery

Enterprise 1.19.x can miss events when multiple event clients are connected. A workaround is available; validate fan-out behavior before relying on the stream for control logic.

### Snowflake key pairs

Snowflake database credential refresh with key-pair authentication has an unresolved 1.19.x failure mode and an available workaround. This is separate from the retirement of password authentication.

## Enterprise lifecycle and licensing

- The 1.19 Enterprise line is an LTS line; 1.16.x moved into long-term support when it became current.
- Vault Enterprise accepts IBM PAO license keys (since 2.0). This license type requires `license_entitlement` in server configuration.
- `common_criteria_mode` is an Enterprise feature flag. It restricts listener cipher suites and enables additional PKI rules; review the cryptography reference before enabling it.

## Safe rollout sequence

1. Record the exact server, CLI, image, and external-plugin patch versions.
2. Remove duplicate HCL attributes and set integrated-storage `disable_mlock` explicitly.
3. Check HSM, seal-wrap, image, plugin-signing, Azure propagation, and event-client hazards.
4. Validate canonical paths, header sizes, JSON limits, and proxy headers against real clients.
5. Test rekey cancellation with nonce retention and confirm all seals before keyring persistence.
6. Upgrade a non-voting or disposable node first where the topology permits it.
7. Confirm health codes, removal state, unseal, Raft membership, plugin registration, auth, rotation, and event flow before proceeding.
