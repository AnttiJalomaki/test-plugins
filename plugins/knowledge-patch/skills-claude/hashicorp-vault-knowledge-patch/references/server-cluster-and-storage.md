# Server, Cluster, and Storage

## Cluster health and Raft membership

- `sys/health` reports whether a node was removed and whether a standby can
  heartbeat the active. Default failure codes are 530 for removed nodes and 474
  for unhealthy HA nodes; override them with `removedcode` and
  `haunhealthycode`.
- `sys/seal-status` and `vault status` expose `removed_from_cluster`. Seal
  status also gains `migration_done_at_epoch`.
- A removed node with existing Raft data is rejected by
  `sys/storage/raft/join`; removed nodes stop serving requests, shut down, and
  seal.
- Seal HA will not persist the barrier keyring unless every seal is healthy.
  Later 1.19 fixes allow new nodes to join Seal-HA clusters.
- With seal wrap enabled, AppRole secrets are seal-wrapped.
  `detect_deadlocks` accepts `sealwrap`, and selected seal-wrap and managed-key
  values can come from environment variables or files.
- A root token can relock a namespace.

## Privileged system operations

- `sys/generate-root`,
  `sys/replication/dr/secondary/generate-operation-token`, and `sys/rekey`
  authenticate callers by default. A primary-generated root token can
  authenticate to a DR secondary.
- Restore legacy unauthenticated behavior only when required:

```hcl
enable_unauthenticated_access = [
  "generate-root",
  "generate-operation-token",
  "rekey",
]
```

- Rekey cancellation requires the operation nonce from 1.19.6 onward.
- The mounts API can unset `allowed_response_headers`.

## Request parsing and forwarding

- Configure `max_json_depth`, `max_json_string_value_length`,
  `max_json_object_entry_count`, and `max_json_array_element_count`. Rate-limit
  quotas are evaluated before JSON limits.
- Vault removes Vault tokens from `Authorization` before forwarding to plugin
  backends unless `Authorization` is explicitly configured as a passthrough
  request header.
- Client addresses from `X-Forwarded-For` must parse as valid IPv4 or IPv6.
- Listeners support `max_token_header_size` for `X-Vault-Token` and bearer
  authorization contents. The default is 8 KB; `-1` disables the limit.
- Canonicalize all API paths. 1.19.19 redirects `/./`, `/../`, and `//` after
  1.19.16 rejected non-canonical paths. A mount tuneable can trim trailing
  slashes on POST. A trailing-slash LIST now honors a more-specific deny
  instead of falling through to a broad allow.

## Integrated storage snapshots and recovery

- Enterprise snapshot recovery can read, list, and recover KV v1 and cubbyhole
  secrets. Later 1.20 patches add database static roles and credentials and the
  SSH plugin CA.
- Send the snapshot ID using `X-Vault-Recover-Snapshot-Id`; the
  `recover_snapshot_id` query parameter is deprecated. `RECOVER` is accepted
  alongside `POST` and `PUT`.
- `vault recover -from` restores an item under a different live path.
- Unload a stuck snapshot with:

```shell
vault operator raft snapshot unload -force
```

  The API equivalent is
  `DELETE sys/storage/raft/snapshot-load/{snapshot_id}?force=true`.
- Automated snapshot configurations support `autoload_enabled`. Generated
  snapshots are loaded automatically for recovery when enabled.
- Snapshot-management and recovery permissions are separate, allowing
  delegated recovery without snapshot-management access.
- Integrated storage rejects `performance_multiplier <= 0`.

## Physical storage and network configuration

- PostgreSQL physical storage can authenticate with AWS IAM, Azure MSI, or GCP
  IAM identities.
- MySQL storage can read `VAULT_MYSQL_USERNAME` and `VAULT_MYSQL_PASSWORD`.
- DynamoDB storage can change its table to per-request billing.
- Raft auto-join can force IPv4 on dual-stack networks.
- Agent, Proxy, server, and other configuration displays canonicalize IPv6 per
  RFC 5952.
- SIGHUP reloads additional Raft settings. The effective values also appear at
  `/sys/config/state/sanitized`.

## Diagnostics and lifecycle

- `pprof-dump-dir` writes startup profile dumps.
- `enable_post_unseal_trace` and `post_unseal_trace_directory` capture
  post-unseal Go traces.
- `vault lease renew --fail-if-not-fulfilled` fails when the requested renewal
  cannot be fulfilled, so chained commands can stop reliably.
- The default API client honors `Retry-After`; rate-limit delays round up to
  whole seconds.
- Enterprise `remove_irrevocable_lease_after` deletes irrevocable leases after
  that duration past expiry. A nonzero setting has a minimum of two days.

## Server-generated state reports

The sudo-protected `sys/reporting/scan` endpoint writes Vault-state report files
to `reporting_scan_directory`. Treat that directory as sensitive operational
output.
