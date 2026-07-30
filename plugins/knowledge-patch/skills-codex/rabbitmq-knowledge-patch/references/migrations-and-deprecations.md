# Migrations and deprecations

This reference covers upgrade sequencing, metadata-store transitions, removed
features, and compatibility cleanup. Batch attribution: `4.0.6`,
`4.1-guides`, `4.1.0`, `4.2-guides`, `4.2.0`, `4.3-guides`, and `4.3.0`.

## Contents

- [Upgrade path planning](#upgrade-path-planning)
- [Feature-flag boot behavior](#feature-flag-boot-behavior)
- [Safe rolling operations](#safe-rolling-operations)
- [Khepri transition](#khepri-transition)
- [Removed partition strategies](#removed-partition-strategies)
- [Queue and feature removals](#queue-and-feature-removals)
- [Removed or ignored configuration](#removed-or-ignored-configuration)
- [Tooling and artifacts](#tooling-and-artifacts)

## Upgrade path planning

### Reaching the 4.1 series

- Upgrade directly from 4.0.x or 3.13.x after enabling every stable feature
  flag.
- A 3.13 cluster with Khepri enabled cannot be upgraded in place because its
  Khepri data format is incompatible with 4.x. Use a blue-green deployment.
- During a rolling upgrade, 4.1.0 and 4.0.x nodes can coexist briefly, but
  4.1-only features remain unavailable until every node is upgraded.

### Reaching the 4.2 series

- Upgrade directly from 4.1.x, 4.0.x, or 3.13.x; an intermediate 4.1 upgrade
  is not required for 4.0 or 3.13.
- `rabbitmqadmin` v2 includes commands intended to automate blue-green
  migrations from 3.13.x to 4.2.x.
- During a rolling upgrade, 4.2.0 nodes can coexist with 4.1.x and 4.0.x
  nodes, but 4.2 features stay disabled until every node runs 4.2.0 or later
  in that series.

### Reaching the 4.3 series

- Upgrade only from 4.2.x, after enabling all stable feature flags. A 3.13.x
  deployment must therefore pass through 4.2.x.
- 4.3.0 and 4.2.x nodes can coexist temporarily during a rolling upgrade.
- If `rabbitmq_amqp1_0` was enabled on 3.13.x and the 4.x cluster still serves
  AMQP 1.0, complete at least one rolling update after enabling
  `rabbitmq_4.0.0` and before moving to 4.3.0.

For all mixed-version states, finish within a few hours. They are upgrade
mechanisms, not supported steady-state topologies.

## Feature-flag boot behavior

Some required feature flags are enabled automatically when a node boots and
every cluster member supports them. The 4.1 series adds no required feature
flags beyond the 4.0.x set.

## Safe rolling operations

Do not use grow-then-shrink to upgrade an entire cluster. It changes replica
identities and can trigger large, unnecessary transfers. Use it only to
replace a single node that must be decommissioned.

Before stopping a node, check whether any quorum queue, stream, or internal
component would lose its online quorum. Automation can wait for quorum plus
one:

```shell
rabbitmq-diagnostics check_if_node_is_quorum_critical
rabbitmq-upgrade await_online_quorum_plus_one
```

After an upgrade, clear management UI cache, local storage, session storage,
and cookies for its domains if stale JavaScript state causes errors.

## Khepri transition

- RabbitMQ 4.2 makes Khepri the default metadata store for new deployments.
  Existing Mnesia deployments stay on Mnesia until an administrator explicitly
  enables Khepri.
- RabbitMQ 4.3 supports only Khepri. Enable the `khepri_db` feature flag before
  upgrading. Otherwise, the first 4.3 node attempts the Mnesia-to-Khepri
  migration during boot.
- Khepri cluster formation uses the same default five-minute timeout as
  Mnesia.
- `rabbitmqctl force_reset` is deprecated because it is incompatible with
  Khepri.
- A reset former Mnesia cluster member now attempts to leave its old cluster
  and retries joining, matching Khepri behavior.
- Third-party plugins should store migratable data in the dedicated directory
  preserved during Mnesia-to-Khepri migration. Other non-whitelisted
  directories under the node data directory can be deleted when migration
  completes.

## Removed partition strategies

The Mnesia-era `pause_if_all_down`, `pause_minority`, and `autoheal` partition
strategies are removed. These settings remain accepted but have no effect and
should be deleted:

- `cluster_partition_handling`
- `cluster_partition_handling.pause_if_all_down.recover`
- `cluster_partition_handling.pause_if_all_down.nodes.$name`

## Queue and feature removals

### Classic queues

- Classic queue v1 storage is removed. A declaration fails when
  `x-queue-version` is `1` or `x-queue-mode` is set to any value.
- Queues converted to CQv2 during a 4.2.x upgrade continue to work.
- Non-durable, non-exclusive classic queues are denied by default. Replace
  them with durable queues, non-durable exclusive queues, or durable queues
  with a queue TTL.
- Temporary compatibility is available with:

```ini
deprecated_features.permit.transient_nonexcl_queues = true
```

STOMP destinations affected by this change now use exclusive queues.

### Other deprecated features

`amqp_address_v1`, `amqp_filter_set_bug`, `global_qos`, and
`queue_master_locator` are denied by default and need explicit opt-in.
`ram_node_type` is removed.

The community `rabbitmq-delayed-message-exchange` plugin is deprecated and
archived. For simple quorum-queue redelivery, use native delayed retries. The
commercial delayed-queue extension covers scheduled routing and delayed
fan-out.

## Removed or ignored configuration

- The etcd peer-discovery keys
  `cluster_formation.etcd.ssl_options.fail_if_no_peer_cert`,
  `cluster_formation.etcd.ssl_options.dh`, and
  `cluster_formation.etcd.ssl_options.dhfile` are unsupported.
- Ineffective `*.cacerts` settings are removed from `rabbitmq.conf`. This does
  not remove or rename `cacertfile`.
- `tcp_listen_options.buffer` is ignored because AMQP user-space TCP buffers
  are auto-tuned. Kernel `recbuf` and `sndbuf` settings are unaffected.
- `rabbitmq-streams set_stream_retention_policy` is a no-op. Configure
  retention with a policy.
- The original all-in-one HTTP API health check no longer performs its former
  comprehensive check. Use focused health endpoints.

## Tooling and artifacts

- Use standalone `rabbitmqadmin` v2, which became generally available in
  4.0.6, instead of the original tool.
- The management plugin no longer serves the v1 download endpoint. If v1 is
  temporarily required, obtain it from the RabbitMQ `v4.2.x` source branch.
- For the complete 4.2.0 source distribution, use
  `rabbitmq-server-4.2.0.tar.xz`, not the automatically generated source
  archive.
- RabbitMQ 4.1 requires at least Erlang/OTP 26.2 and supports the Erlang/OTP
  27.x series.
