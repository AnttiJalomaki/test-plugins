# Upgrades and Lifecycle

## Support and upgrade cadence

Consul 1.21 is an Enterprise long-term support release with two years of
support (1.21.0). Operators can remain on it while receiving patches and fixes.
The 2.0 version number adds Enterprise support options for longer contract
periods; earlier Enterprise releases remain maintained under their existing
long-term-support contracts (2.0.0).

Routine upgrades should cross at most two major Consul version jumps. For
example, if `1.21.x` is current, start no earlier than `1.19.x`. Community
operators need to move to the latest major release about every four months.
Standard Enterprise major releases are maintained for about one year.
Enterprise LTS operators can upgrade about annually, jump at most three major
versions, and expect about two years of maintenance for an LTS release.

## Enterprise license transition

When upgrading to Consul 1.21.7+ent or later with the updated license, use this
order (1.21-upgrade):

1. Upgrade server agents before client agents.
2. Update the license on servers and clients to `enterprise-standard`.
3. Restart only the server agents, one at a time.
4. Leave clients on the existing license while the server set becomes ready.
5. Restart clients with the new license after the servers are ready.

## Standard rolling upgrade

Restart servers on the new Consul version one at a time, waiting for each to
become healthy and rejoin before continuing. Roll clients only after the server
set is complete. Run `consul members` to verify every agent's build and
protocol.

For a service-mesh client:

1. Stop the old Consul agent.
2. Stop its associated Envoy proxies.
3. Start the new Consul agent.
4. Restart Envoy proxies on a version compatible with the new agent.

WAN-federated service-mesh clients should use centralized sidecar and
mesh-gateway configuration:

```hcl
enable_central_service_config = true
```

Consul 1.8.4 or later reports compatible Envoy versions from
`/v1/agent/self`. If both the old and new Consul versions support the installed
Envoy version, Envoy does not need to move immediately.

## Incompatible protocol transitions

When the release notes require an incompatible protocol transition, split the
work into two phases.

First, run the new binary while speaking the previous protocol. Restart servers
one at a time and wait for each rejoin, then update the rest of the agents.

```shell
consul -v
consul agent -protocol=PREVIOUS
```

After every node runs the new binary, restart every agent without the protocol
override. `-protocol` changes only the protocol version the agent speaks, not
the complete range it understands. Keeping the older spoken protocol can
disable newer features.

## WAN-federated rollout

Upgrade the primary datacenter's servers and then its clients. Repeat that
server-then-client sequence for each secondary datacenter.

Within every server set:

1. Identify the Raft leader with `consul operator raft list-peers`.
2. Upgrade followers first and the leader last.
3. For each server, run `consul leave` and wait for state `left`.
4. Start the new binary and wait for state `alive` before continuing.

This sequence preserves quorum.

## Availability and ACL recovery

From `consul leave` until a client agent restarts, its services are unhealthy
and undiscoverable. Zero downtime requires redundant service instances on
other clients.

If `enable_token_persistence` was not enabled and a server's tokens are not in
its configuration files, reapply the `agent` and `default` tokens after the
restart. The server cannot rejoin until those tokens are restored.

After every datacenter is upgraded, verify WAN membership:

```shell
consul members -wan
```

Then query `/v1/acl/replication` through an agent in a secondary datacenter:

```shell
curl -s -H "X-Consul-Token: $CONSUL_HTTP_TOKEN" \
  "https://consul-server-0.secondary/v1/acl/replication?pretty"
```

The primary datacenter always reports ACL replication as disabled, even when
replication is working. Verify from a secondary.

## Automated server replacement

Consul Enterprise automated upgrades are enabled by default when
`DisableUpgradeMigration` is `false`. Confirm or enable the setting with:

```shell
consul operator autopilot get-config
consul operator autopilot set-config -disable-upgrade-migration=false
```

New-version servers initially join as non-voters. After enough replacements
join to form a quorum, Autopilot promotes them, transfers leadership to the new
set, and demotes old servers. Remove the old servers with `consul leave`.
The two Consul versions must support a direct upgrade.

## Version-tagged infrastructure migrations

For server replacement caused by an image, operating system, or configuration
change rather than a Consul binary change, point `UpgradeVersionTag` at a
`node_meta` key. Autopilot compares the key's semantic versions in `X`, `X.Y`,
or `X.Y.Z` form instead of comparing Consul versions.

Tag existing servers with the old value and reload them before joining
replacement servers with a newer value:

```hcl
node_meta {
  build = "0.0.2"
}
autopilot {
  upgrade_version_tag = "build"
}
```
