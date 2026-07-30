---
name: nomad-knowledge-patch
description: HashiCorp Nomad
version: 2.0.0
license: MIT
metadata:
  author: Nevaberry
---



# HashiCorp Nomad Knowledge Patch

Use this skill when upgrading, configuring, extending, or troubleshooting
Nomad servers, clients, jobs, task drivers, storage, ACLs, identity, or
operational tooling.

## Working Method

1. Determine the exact server, client, CLI, and Go API versions in use. In a
   mixed cluster, record the version of each agent class and region.
2. Identify the affected surface: cluster upgrade, server configuration,
   security, job scheduling, storage, driver/plugin behavior, or CLI/API
   automation.
3. Read every relevant topic reference before changing an option, jobspec,
   policy, query, alert, or custom integration.
4. Treat removals, ignored fields, changed defaults, and validation changes as
   migration work rather than optional feature adoption.
5. Prefer the deployed configuration, jobs, plugin code, API types, tests, and
   observed runtime behavior when local compatibility patches differ.
6. If the installed release is newer than this skill, say that the guidance
   may be stale and verify the affected behavior against the deployed binary.

## Reference Index

| Reference | Topics |
| --- | --- |
| [upgrading-and-server-operations.md](references/upgrading-and-server-operations.md) | Upgrade order, version skew, downgrades, health checks, Raft protocol and WAL migration, joins, startup, licensing |
| [security-identity-and-governance.md](references/security-identity-and-governance.md) | ACL behavior, workload and client identity, OIDC, job secrets, Consul and Vault authentication, Sentinel |
| [storage-and-volumes.md](references/storage-and-volumes.md) | Dynamic host volumes, governance, CSI visibility, scheduling capacity, quota schema |
| [jobs-scheduling-and-deployments.md](references/jobs-scheduling-and-deployments.md) | System jobs, deployment behavior, updates, interpolation, placement limits, networking |
| [drivers-plugins-and-platforms.md](references/drivers-plugins-and-platforms.md) | Plugin activation, removed driver interfaces, Docker, raw_exec, QEMU, executor failures, architecture support |
| [cli-api-and-observability.md](references/cli-api-and-observability.md) | CLI flags and UI links, Go API migrations, evaluation diagnostics, event streams, metrics |

## Highest-Priority Upgrade Checks

### Migrate server storage and join configuration

- Replace `raft_boltdb` with `raft_logstore`.
- Move `server.retry_join`, `server.retry_interval`, `server.retry_max`, and
  `server.start_join` into `server.server_join`.
- Prepare callers of `nomad server join` and the Join Agent API to authenticate
  with a token carrying `agent:write`.
- Treat `nomad operator raft migrate-backend` as an irreversible in-place move
  to WAL. Keep a pre-migration snapshot if a return to BoltDB might be needed.
- Run join commands against the region leader, or the authoritative region
  when federating a region. Prefer `server_join` with gossip encryption and
  mTLS for a new cluster.

### Remove or replace rejected and ignored jobspec behavior

- Remove `reschedule` blocks from `system` and `sysbatch` jobs; submission now
  fails instead of ignoring them.
- Rename any task called `alloc`; that name is reserved because it breaks
  inter-task filesystem isolation.
- Replace deprecated task-group disconnect fields with the `disconnect` block.
  The old fields no longer have an effect.
- Do not depend on a template block to create a Consul identity implicitly.
- Remove legacy token-based allocation authentication for both Consul and
  Vault.

### Update custom plugins and integrations

- Add an explicit matching `plugin` configuration block for each executable in
  `plugin_dir`; unconfigured executables are skipped.
- Remove custom task-driver use of remote tasks; that interface is gone.
- Correct duplicate or invalid ACL policy keys before attempting to write the
  policy again.
- Replace `QuotaSpec.VariablesLimit` with
  `QuotaSpec.RegionLimit.Storage.Variables`, and account for
  `QuotaSpec.RegionLimit` using `QuotaResources`.
- Replace `Node.Resources` and `Node.Reserved` with `Node.NodeResources` and
  `Node.ReservedResources`; the deprecated fields are never populated.

## Safe Cluster Upgrade Procedure

### Upgrade servers before clients

Upgrade one server at a time. After each server rejoins:

1. Check membership with `nomad server members`.
2. Compare its `nomad agent-info` `last_log_index` with the other servers.
3. Continue only after replication is current.

Choose a shutdown signal that does not activate `leave_on_terminate` or
`leave_on_interrupt`. For example, when `leave_on_terminate` is enabled, use
`SIGINT` rather than `SIGTERM` for an in-place restart.

### Protect client allocations

- A client restart longer than the configured `heartbeat_grace` can cause all
  allocations on that node to be rescheduled.
- Drain an old client when replacing it instead of upgrading it in place.
- Do not plan an in-place downgrade. A downgraded client requires drained
  allocations and removal of its data directory; a safe server downgrade
  requires reprovisioning the cluster.
- Delay use of new features until all relevant agents are upgraded. For
  federation, include every agent in the region and the authoritative region's
  servers.

### Handle Raft migrations deliberately

For a multi-server Raft protocol migration, stop and force-leave one server at
a time, verify `RaftProtocol` with `nomad operator raft list-peers`, confirm
replication, and leave the leader until last. A single server needs a
new-format `server/raft/peers.json` before its protocol restart; read the full
procedure before attempting it.

## Security and Identity Quick Reference

### Choose modern authentication paths

- OIDC auth methods can use a private-key JWT client assertion instead of a
  client secret.
- Enable PKCE with `OIDCEnablePKCE: true`; it works with either authentication
  method, but the provider must support and may need to enable PKCE.
- Introduce clients with signed JWT introduction tokens when node-name,
  node-pool, or TTL constraints are required. Servers then issue and rotate a
  client identity for RPC authentication alongside mTLS.
- Use a jobspec `secret` block to fetch from Nomad, Vault, or a custom
  secret-provider plugin, then interpolate `${secret.secret_name.key}`.

### Account for ACL behavior changes

- `/v1/acl/token/self` distinguishes disabled ACLs from missing credentials:
  disabled ACLs return `200` with an explanatory body, while enabled ACLs
  without a valid token return `403`.
- Existing policies with duplicate or invalid keys continue to operate, but
  cannot be written again until their source is fixed.
- Workload identity tokens can list and retrieve policies through the ACL API.
- `nomad sentinel apply` requires `-scope`.

## Jobs and Scheduling Quick Reference

### Control scale and rollout behavior

- `job_max_count` limits the sum of a job's task-group `count` values at
  submission or scale time. Its default is `50000`; changing it does not
  retroactively affect existing jobs.
- System jobs support deployments and their `update` block can drive canary or
  blue/green rollouts.
- For service and batch jobs, changing a task group to `count = 0` behaves as
  removal and stops its non-terminal allocations.
- Use the job-update CLI's `-preserve-resources` option when updating a job
  while retaining its existing resource block.

### Recheck update and interpolation assumptions

- Affinity and spread changes are not destructive updates.
- Nomad-native services interpolate correctly during in-place updates.
- Task-level services, checks, and identities do not interpolate jobspec values
  from other tasks in the group.
- Consul Connect permits `cni/*` network modes, but treat the combination as
  experimental-risk behavior.

## Storage Quick Reference

- Dynamic host volumes can be created through the CLI or API without a client
  restart and consumed by stateful deployments.
- Nomad schedules their declared availability but does not interpret the
  backing storage; distinguish node-local storage from highly available
  network storage in operational design.
- Storage available for scheduling is calculated as total bytes minus
  `client.reserved.disk`, not free disk space. Reserve space for the host
  operating system.
- Use `region_limit.storage.variables` rather than the deprecated
  `variables_limit` quota field.
- Enterprise volume creation can apply Sentinel policy, namespace capacity
  quotas, and namespace node-pool validation.

## Drivers, CLI, and Observability Quick Reference

- Set `telemetry.publish_allocation_metrics = true` explicitly on clients that
  must export allocation metrics.
- Update broker metric queries for parent job IDs and removal of the
  `eval_id` label from the evaluation-waiting metric.
- QEMU filesystem environment variables are host paths; do not assume `/alloc`
  or `/local` container-style paths.
- Executor-launch failures in `exec`, `raw_exec`, `java`, and `qemu` report
  exit code `-1`.
- Use `-group` with `nomad alloc exec`, `nomad alloc logs`, or
  `nomad alloc fs` when selecting a task group.
- Disable CLI web UI hints with `ui.show_cli_hints = false` or
  `NOMAD_CLI_SHOW_HINTS=0`; use `-ui` to open a generated link.
- Read the detailed evaluation and placement output before building separate
  scheduling diagnostics around older terse responses.
