---
name: nomad-knowledge-patch
description: HashiCorp Nomad
version: 2.0.0
license: MIT
metadata:
  author: Nevaberry
---


# HashiCorp Nomad Knowledge Patch

Use this skill when reviewing, upgrading, configuring, or operating Nomad
clusters and jobs. Start with the project's pinned Nomad version and apply only
guidance introduced at or below that version. Prefer the actual configuration,
jobspecs, API types, and observed cluster behavior when they disagree with this
guidance.

## Reference index

| Reference | Topics |
| --- | --- |
| [upgrades-and-raft.md](references/upgrades-and-raft.md) | Upgrade order, skew, downgrade limits, server health, Raft protocol and WAL migrations |
| [jobs-and-scheduling.md](references/jobs-and-scheduling.md) | Jobspec compatibility, deployments, updates, placement, limits, networking |
| [identity-and-policy.md](references/identity-and-policy.md) | ACLs, workload and client identities, OIDC, Consul and Vault authentication, Sentinel |
| [storage-drivers-and-plugins.md](references/storage-drivers-and-plugins.md) | Dynamic host volumes, storage accounting, task drivers, external and secrets plugins |
| [operations-and-observability.md](references/operations-and-observability.md) | Agent settings, CLI behavior, API changes, metrics, events, licensing, platform support |

## Breaking changes to check first

### Validate jobs before submission

- Remove `reschedule` blocks from `system` and `sysbatch` jobs. They are
  rejected instead of ignored.
- Rename any task called `alloc`; the name is reserved because it breaks
  inter-task filesystem isolation.
- Replace deprecated task-group disconnect fields with the `disconnect` block.
  The old fields are ignored.
- Custom task drivers must not use remote tasks; that interface was removed.
- Do not expect a `template` block to grant a Consul identity implicitly.
- For system jobs, use the deployment created from the `update` block to
  control canary or blue/green rollout.

### Make plugins and policy explicit

- Every executable in `plugin_dir` needs a matching `plugin` configuration
  block. Unconfigured executables are skipped.
- `nomad sentinel apply` requires `-scope`.
- Correct duplicate or invalid ACL policy keys before rewriting a policy.
  Existing affected policies can continue working, but rejected source cannot
  be saved again unchanged.
- Replace removed Consul and Vault token-based allocation authentication with
  workload-identity-based flows.

### Migrate server configuration

- Replace `raft_boltdb` with `raft_logstore`.
- Replace `server.retry_join`, `server.retry_interval`, `server.retry_max`, and
  `server.start_join` with `server.server_join`.
- Plan authenticated manual joins. A token with `agent:write` becomes required;
  run a regional join against the region leader and a federation join against
  the authoritative region.
- For new clusters, prefer `server_join` together with gossip encryption and
  mTLS instead of manual joins.

### Update API consumers

- Replace quota `variables_limit` and Go API `QuotaSpec.VariablesLimit` with
  `region_limit.storage.variables` and
  `QuotaSpec.RegionLimit.Storage.Variables`.
- Account for `QuotaSpec.RegionLimit` changing from `Resources` to
  `QuotaResources`.
- Replace deprecated, unpopulated `Node.Resources` and `Node.Reserved` fields
  with `Node.NodeResources` and `Node.ReservedResources`.
- Handle `/v1/acl/token/self` returning `200` when ACLs are disabled and `403`
  when ACLs are enabled but the request lacks a valid token.

## Upgrade safely

1. Inspect the target Enterprise license before upgrading an Enterprise server
   when applicable.
2. Upgrade servers one at a time, leaving the leader until last for Raft
   protocol changes.
3. After each server returns, verify membership and wait for its
   `last_log_index` to catch up before continuing.
4. Upgrade clients after servers. Keep restarts inside `heartbeat_grace`, or
   drain clients before replacement to avoid allocation rescheduling.
5. Delay use of new features until all relevant agents are upgraded, including
   authoritative-region servers in federated deployments.
6. Take a snapshot before migrating the Raft backend from BoltDB to WAL. The
   backend cannot be changed back in place.

Nomad aims for compatibility across at least two point releases, but
downgrades are unsupported. A client downgrade requires draining allocations
and deleting its data directory. A safe server downgrade requires
reprovisioning the cluster.

For exact shutdown signals, force-leave handling, Raft protocol 3 procedures,
single-server `peers.json`, and WAL recovery implications, load
[upgrades-and-raft.md](references/upgrades-and-raft.md).

## Adopt high-value capabilities

### Dynamic host volumes

Create host volumes without restarting clients:

```shell
nomad volume create ./internal-plugin.volume.hcl
```

Jobs consume them with `volume` and `volume_mount` blocks. The scheduler tracks
availability, but it does not understand whether the backing storage is local
or highly available. Enterprise deployments can add Sentinel checks,
per-namespace capacity quotas, and node-pool validation.

### Jobspec secrets

Use the `secret` block to fetch a value from Nomad, Vault, or a custom
secret-provider plugin, then interpolate it as:

```hcl
${secret.secret_name.key}
```

Allow for the 60-second secrets-plugin execution timeout when diagnosing slow
providers.

### Client introduction and identity

Generate a constrained, signed introduction token and supply it when the
client first joins:

```shell
nomad node intro create
nomad agent -client-intro-token=<token>
```

Servers can enforce token constraints for node name, node pool, and TTL, and
emit violation logs and metrics. After registration, the server issues and
rotates a client identity for RPC authentication alongside mTLS:

```shell
nomad node identity get
nomad node identity renew
```

### System deployments and zero-count groups

System jobs support deployments driven by `update`, including canary and
blue/green strategies. Inspect them in the web UI or with
`nomad deployment` commands. For service and batch jobs, changing a task group
to `count = 0` behaves like removing it and stops its non-terminal allocations.

## Configuration quick reference

Set a longer server startup allowance when keyring decryption or other setup
cannot finish within the default `30s`:

```hcl
server {
  start_timeout = "1m"
}
```

Allocation metrics are a true opt-in. Set this on clients that must publish
them:

```hcl
telemetry {
  publish_allocation_metrics = true
}
```

Limit the sum of task-group counts accepted for a newly submitted or scaled
job:

```hcl
server {
  job_max_count = 100000
}
```

The default is `50000`; changing it does not retroactively affect existing
jobs.

Storage capacity for placement is calculated from total bytes minus
`client.reserved.disk`, not current free space. Reserve space used by the host
operating system and stop depending on `unique.storage.bytesfree`.

`num_schedulers` must be between zero and the machine's available CPU count.

## CLI and diagnostic quick reference

- `nomad alloc exec`, `nomad alloc logs`, and `nomad alloc fs` accept `-group`.
- Job updates accept `-preserve-resources` to retain the existing resource
  block.
- `nomad eval status` exposes related evaluations, allocations, annotations,
  failed placements, and preemptions; use it before assuming a scheduler bug.
- `nomad alloc status -verbose` includes evaluated and rejected node counts and
  node scores.
- Common commands print web UI hints and accept `-ui`. Disable hints with
  `ui.show_cli_hints = false` or `NOMAD_CLI_SHOW_HINTS=0`.
- `nomad volume status` shows capabilities, while `nomad volume delete`
  accepts an ID prefix and wildcard namespace.

## Behavioral checks

- Affinity and spread edits are non-destructive.
- During in-place updates, Nomad-native services interpolate correctly.
  Task-level services, checks, and identities do not interpolate jobspec values
  from other tasks in the group.
- Executor failures in `exec`, `raw_exec`, `java`, and `qemu` report exit code
  `-1`.
- QEMU filesystem environment variables are host paths. Do not treat `/alloc`
  or `/local`-style relative container paths as the current behavior.
- Consul Connect with `cni/*` networking is permitted but explicitly
  use-at-your-own-risk.
- Variable, CSI volume, and CSI plugin activity can be consumed from the event
  stream rather than polled.

Load the topic references before changing production configuration or
implementing API and jobspec migrations; they preserve the version-sensitive
details and exceptions behind these quick checks.
