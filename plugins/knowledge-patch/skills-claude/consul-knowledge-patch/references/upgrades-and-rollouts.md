# Upgrades and Rollouts

## Choose an Upgrade Cadence

Keep a routine Community edition upgrade within two major Consul jumps. For
example, if `1.21.x` is current, start from no earlier than `1.19.x`.
Community operators generally need the latest major about every four months.

Enterprise LTS operators can generally upgrade annually and jump at most three
major versions. Standard Enterprise major releases are maintained for about
one year, while LTS releases are maintained for about two years. Treat direct
upgrade compatibility as a separate requirement from maintenance status.

## Roll Servers, Clients, and Envoy

Restart server agents on the new Consul version one at a time. Wait for each
server to become healthy and rejoin before proceeding. Roll client agents only
after the server set is complete, and use `consul members` to confirm every
agent's build and protocol.

On a service-mesh client:

1. Stop the old Consul agent.
2. Stop its associated Envoy proxies.
3. Start the new Consul agent.
4. Restart Envoy proxies that are compatible with the new agent.

Prepare WAN-federated service-mesh clients by enabling centralized sidecar and
mesh-gateway configuration:

```hcl
enable_central_service_config = true
```

Consul 1.8.4 or later can report compatible Envoy versions through
`/v1/agent/self`. If the old and new Consul versions both support the installed
Envoy version, an immediate Envoy upgrade may be unnecessary.

## Handle Incompatible Protocol Transitions

When the applicable release notes require an incompatible protocol
transition, use two phases:

1. Start the new binary while forcing the previous spoken protocol.
2. Restart servers one at a time and wait for each to rejoin.
3. Complete the binary rollout across every server and client.
4. Restart every agent without the protocol override.

```shell
consul -v
consul agent -protocol=PREVIOUS
```

The `-protocol` option changes only the protocol version an agent speaks, not
the complete range it understands. Speaking an older protocol can disable
newer features, so remove the override only after all nodes run the new binary.

## Roll WAN-Federated Datacenters

Upgrade the primary datacenter's servers and then its clients. Repeat the
server-then-client order for each secondary datacenter.

Within each server set:

1. Run `consul operator raft list-peers` to identify the leader.
2. Upgrade followers before the leader.
3. Run `consul leave` for one server and wait until its status is `left`.
4. Start the new binary and wait for the server to become `alive`.
5. Continue only after health and quorum are intact.

This sequence preserves quorum during the rollout.

After every datacenter is upgraded, run `consul members -wan`. Then query
`/v1/acl/replication` through an agent in a secondary datacenter:

```shell
curl -s -H "X-Consul-Token: $CONSUL_HTTP_TOKEN" \
  "https://consul-server-0.secondary/v1/acl/replication?pretty"
```

Do not use the primary datacenter's replication status as the verification
signal: the primary always reports ACL replication as disabled even when
replication is functioning.

## Protect Client Availability and Restore Tokens

A client agent's services are unhealthy and undiscoverable from
`consul leave` until the agent restarts. Provide redundant service instances
on other clients for zero-downtime upgrades.

If `enable_token_persistence` was disabled and the server's configuration
files do not contain its tokens, reapply both the `agent` and `default` tokens
after restart. The server cannot rejoin until those tokens are restored.

## Update Enterprise Licenses Safely

For an upgrade to Consul 1.21.7+ent or later using the updated license
(batch `1.21-upgrade`), upgrade server agents before client agents. Update the
license on both roles to `enterprise-standard`, but initially restart only the
servers, one at a time. Clients continue using the existing license while
servers become ready. Restart clients with the new license only after the
server rollout succeeds.

## Automate Server-Set Replacement

Enterprise automated upgrades use Autopilot upgrade migration. The feature is
enabled by default when `DisableUpgradeMigration` is `false`.

New-version servers first join as non-voters. When enough new servers exist to
form quorum, Autopilot promotes them, transfers leadership to the new set, and
demotes the old servers. Remove the demoted servers with `consul leave`.
Automatic replacement does not waive compatibility: the old and new Consul
versions must support a direct upgrade.

```shell
consul operator autopilot get-config
consul operator autopilot set-config -disable-upgrade-migration=false
```

## Migrate by Node Version Tag

Use `UpgradeVersionTag` to replace servers for an image, operating-system, or
configuration change without changing the Consul binary. Set it to the name of
a `node_meta` key. Autopilot compares semantic values shaped as `X`, `X.Y`, or
`X.Y.Z` instead of comparing Consul versions.

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

## Interpret Enterprise Support Lines

Consul 1.21 is an Enterprise LTS release with two years of patches and fixes
(since 1.21.0). The 2.0 version number adds Enterprise options for longer
contract periods (since 2.0.0). Earlier Enterprise releases continue under
their existing LTS contracts; do not infer that the new major number changes
those agreements.
