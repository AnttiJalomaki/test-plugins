# Upgrades and Deprecations

Use this reference when selecting an upgrade path, preparing feature flags,
migrating metadata, or removing behavior that a newer node no longer honors.

## Upgrade Paths and Rolling Operation

### Moving to 4.1

The guidance in batch `4.1-guides` permits a direct upgrade to 4.1 from 4.0.x
or 3.13.x after all stable feature flags are enabled. A 3.13 cluster with
Khepri enabled is the exception: its Khepri format is incompatible with 4.x,
so use a blue-green deployment instead of an in-place upgrade.

Nodes on 4.1.0 can temporarily coexist with 4.0.x nodes. Features specific to
4.1 remain unavailable until every node is upgraded. Mixed-version operation
is only an upgrade mechanism and should last no more than a few hours.

Do not use grow-then-shrink to upgrade an entire cluster. It changes replica
identities and can cause large, unnecessary data transfers. It remains
appropriate for replacing one node that must be decommissioned.

Before stopping a node, determine whether its loss would remove the online
quorum of a quorum queue, stream, or internal component:

```shell
rabbitmq-diagnostics check_if_node_is_quorum_critical
rabbitmq-upgrade await_online_quorum_plus_one
```

The second command allows automation to wait for quorum-plus-one.

Some required feature flags are automatically enabled at boot once every
cluster node supports them. RabbitMQ 4.1 adds no required flags beyond the
4.0.x set.

RabbitMQ 4.1 requires Erlang/OTP 26.2 or later and supports Erlang/OTP 27.x.

After an upgrade, stale management UI JavaScript can produce errors. Clear
browser cache, local storage, session storage, and cookies for the management
UI domains.

### Moving to 4.2

The direct paths in batch `4.2-guides` allow 4.2 to be reached from 4.1.x,
4.0.x, or 3.13.x; no intermediate 4.1 upgrade is required.

Khepri is the default metadata store for new 4.2 deployments. An upgraded
deployment already using Mnesia remains on Mnesia until an administrator
explicitly enables Khepri.

`rabbitmqadmin` v2 supplies commands intended to automate blue-green
migrations from 3.13.x to 4.2.x.

During a Mnesia-to-Khepri migration, third-party plugins can store preserved
data in the dedicated plugin directory. Non-whitelisted directories inside
the node data directory can be deleted when migration completes.

Nodes on 4.2.0 can temporarily coexist with 4.1.x and 4.0.x nodes, but
4.2-specific features remain unavailable until all nodes run 4.2.0 or later
in that series. Finish the rolling upgrade within a few hours.

For a complete 4.2.0 source distribution, use
`rabbitmq-server-4.2.0.tar.xz`, not GitHub's automatically generated source
archive.

### Moving to 4.3

The guidance in batch `4.3-guides` requires upgrading to 4.3.x from 4.2.x,
with all stable feature flags enabled. A 3.13.x deployment must therefore
upgrade through 4.2.x.

Khepri is the only supported metadata store in 4.3. Enable the `khepri_db`
feature flag before upgrading. If it is not enabled, the first 4.3 node
migrates Mnesia metadata during boot.

Batch `4.3.0` permits 4.3.0 nodes to coexist temporarily with 4.2.x nodes
during a rolling upgrade. Do not leave the cluster mixed for more than a few
hours.

If a cluster had `rabbitmq_amqp1_0` enabled on 3.13.x and continues to serve
AMQP 1.0 on 4.x, complete at least one rolling update after enabling
`rabbitmq_4.0.0` and before upgrading to 4.3.0.

## Removed, Denied, Ignored, and Deprecated Behavior

### Metadata and cluster administration

`rabbitmqctl force_reset` is deprecated since batch `4.1.0` because it is
incompatible with Khepri.

In 4.3, the Mnesia-era partition strategies `pause_if_all_down`,
`pause_minority`, and `autoheal` are removed.
`cluster_partition_handling`,
`cluster_partition_handling.pause_if_all_down.recover`, and
`cluster_partition_handling.pause_if_all_down.nodes.$name` are still accepted
but do nothing and should be removed.

The deprecated `ram_node_type` feature is removed. The deprecated features
`amqp_address_v1`, `amqp_filter_set_bug`, `global_qos`, and
`queue_master_locator` are denied by default and require explicit opt-in.

### Queue declarations

RabbitMQ 4.3 rejects non-durable, non-exclusive classic queues by default.
Use durable queues, non-durable exclusive queues, or durable queues with a
queue TTL. A temporary compatibility switch is available:

```ini
deprecated_features.permit.transient_nonexcl_queues = true
```

Classic queue v1 storage is removed. A classic queue declaration fails if
`x-queue-mode` is set to any value or `x-queue-version` is `1`. Classic queues
already converted to CQv2 during a 4.2.x upgrade continue to work.

### Configuration removals and ignored settings

The 4.1 etcd peer-discovery settings
`cluster_formation.etcd.ssl_options.fail_if_no_peer_cert`,
`cluster_formation.etcd.ssl_options.dh`, and
`cluster_formation.etcd.ssl_options.dhfile` are unsupported.

AMQP listener user-space TCP buffers are auto-tuned in 4.1, so
`tcp_listen_options.buffer` is ignored. This does not affect kernel-level
`recbuf` and `sndbuf`.

The ineffective `*.cacerts` keys are removed from `rabbitmq.conf` in batch
`4.2.0`. The valid `cacertfile` setting is neither removed nor renamed.

### Tooling and health checks

The original HTTP API all-in-one health check no longer performs its former
mega-check as of batch `4.0.6`; use focused checks.

`rabbitmq-streams set_stream_retention_policy` no longer changes retention.
Configure retention with a policy.

The management plugin no longer exposes the `rabbitmqadmin` v1 download
endpoint. Prefer `rabbitmqadmin` v2; if v1 is unavoidable, obtain it from the
RabbitMQ `v4.2.x` source branch.
