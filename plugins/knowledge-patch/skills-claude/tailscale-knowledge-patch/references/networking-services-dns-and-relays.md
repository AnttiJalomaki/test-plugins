# Networking, Services, DNS, and Relays

## Host and consume Tailscale Services

Tailscale Services are generally available, and `tsnet` nodes can host them
(1.94.1). Clients on every platform automatically accept Service virtual IPs
regardless of `--accept-routes`; Operator egress proxies can send traffic to
those VIPs. A Tailscale Service can use a remote target as its service
destination (1.92.1).

Services advertise automatically when Tailscale starts (1.96.2). Disable only
when the deployment needs explicit advertisement control:

```text
TS_EXPERIMENTAL_SERVICE_AUTO_ADVERTISEMENT=false
```

Tailscale Serve and Funnel can send a PROXY protocol header before proxied
traffic so the destination receives the original client's source IP address
and port (1.92.1).

## Operate exit nodes and subnet routers

Use `auto:any` to track the recommended exit node and switch automatically as
available nodes or network conditions change. Linux, Windows, and macOS accept
the CLI setting, while Windows, macOS, iOS, and tvOS expose a Recommended
picker option (1.86.0):

```console
tailscale up --exit-node=auto:any
tailscale set --exit-node=auto:any
```

On Windows and macOS, `ExitNode.AllowOverride` can be combined with
`ExitNodeID=auto:any` to require exit-node use while permitting the user to
choose another node (1.88.1). An iOS device can itself serve as an exit node
(1.98.1).

Android devices can be configured as subnet routers from the app's Settings
menu (1.80.0). iOS also provides a subnet-routing toggle. Version 1.84.1 fixes
the unintended default-off state for subnet routing on both iOS and Android
(1.84.0).

Linux reports a health check when IP forwarding is misconfigured for a subnet
router or exit node. It also sets `src_valid_mark` together with its `connmark`
firewall rules so reverse-path filtering does not discard routed traffic
(1.98.1).

## Resolve DNS through the tailnet

- While using an exit node, clients can still send all domains to DNS
  resolvers configured in the admin console's DNS nameserver settings
  (1.90.1).
- Linux, Windows, and macOS correctly use DNS-over-TCP fallback when an
  upstream resolver is reachable only through the tailnet (1.84.0).
- On Linux, MagicDNS resolves through `resolv.conf` without requiring a DNS
  manager (1.94.1).
- `tailscale dns query` and `tailscale dns status` support `--json` for
  machine-readable results (1.96.2):

  ```console
  tailscale dns status --json
  ```

The withdrawn Linux v1.98.1 build has a MagicDNS interaction regression; do
not mistake that release fault for a DNS configuration failure (1.98.1).

## Configure DERP servers

Clients can pin a self-signed IP-address certificate for DERP, supporting
deployments that cannot use Let's Encrypt or another WebPKI certificate
(1.82.0). On Linux, a custom DERP server can instead use Google Cloud Platform
Certificate Manager (1.94.1).

Singapore DERP servers changed both IPv4 and IPv6 addresses. Only deployments
with custom firewall rules pinned to the old addresses need action: read the
current addresses from the DERP map and update those rules (1.88.1).

## Configure and monitor Peer Relays

Assign fixed endpoints to a Peer Relay with (1.92.1):

```console
tailscale set --relay-server-static-endpoints=<endpoint-list>
```

Peer Relays can advertise addresses discovered through Amazon EC2 Instance
Metadata Service (1.96.2). Observe their forwarding with
`tailscaled_peer_relay_forwarded_packets_total` and
`tailscaled_peer_relay_forwarded_bytes_total` (1.94.1), and observe advertised
endpoint count with the `tailscaled_peer_relay_endpoints` user-metric gauge
(1.96.2).
