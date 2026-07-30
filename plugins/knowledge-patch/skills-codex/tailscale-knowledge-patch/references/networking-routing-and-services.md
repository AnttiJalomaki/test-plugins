# Networking, Routing, and Services

## Coordination and DERP

- Custom coordination servers using HTTP with an explicit custom port are
  accepted (since 1.80.0).
- Clients can pin self-signed certificates for DERP servers addressed by IP,
  supporting deployments that cannot use Let's Encrypt or another WebPKI
  certificate (since 1.82.0).
- Singapore DERP servers changed IPv4 and IPv6 addresses in the 1.88.1 era.
  Deployments that pin addresses in firewall rules must refresh them from the
  DERP map; other deployments need no action.
- On Linux, custom DERP servers can use Google Cloud Platform Certificate
  Manager (since 1.94.1).

## DNS through tailnet routes and exit nodes

- Linux, Windows, and macOS correctly use DNS-over-TCP fallback when the
  upstream resolver is reachable only through the tailnet (since 1.84.0).
- Clients using an exit node can still send all domains to resolvers configured
  in the admin console's DNS nameserver settings (since 1.90.1).
- On Linux, MagicDNS works with `resolv.conf` even when no DNS manager is in
  use (since 1.94.1).

## Exit nodes and Linux routing

- Linux, Windows, and macOS can track the recommended exit node with
  `auto:any` (since 1.86.0):

  ```console
  tailscale up --exit-node=auto:any
  tailscale set --exit-node=auto:any
  ```

  The selection changes with node availability or network conditions.
  Windows, macOS, iOS, and tvOS expose the behavior through the Recommended
  picker option.
- Linux reports bad IP forwarding for subnet routers and exit nodes as a
  health check (since 1.98.1). It also sets `src_valid_mark` beside `connmark`
  firewall rules so reverse-path filtering does not drop routed packets.

## App connectors, Serve, and Funnel

- App connectors are generally available as of 1.84.0 for securing tailnet
  access to SaaS applications.
- Serve and Funnel can send a PROXY protocol header before proxied traffic so
  the destination receives the original client's source IP and port (since
  1.92.1).

## Tailscale Services

- A Tailscale Service can point at a remote service destination (since
  1.92.1).
- Tailscale Services are generally available as of 1.94.1, and `tsnet` nodes
  can host them.
- Clients on every platform accept Service virtual IPs regardless of
  `--accept-routes`. Operator egress proxies can send traffic to Service VIPs.
- Services advertise automatically at startup in 1.96.2. To retain manual
  advertisement, disable the default:

  ```text
  TS_EXPERIMENTAL_SERVICE_AUTO_ADVERTISEMENT=false
  ```

## Peer Relay

- Configure static Peer Relay endpoints with
  `tailscale set --relay-server-static-endpoints` (since 1.92.1).
- Peer Relays advertise addresses discovered from the Amazon EC2 Instance
  Metadata Service (since 1.96.2).
