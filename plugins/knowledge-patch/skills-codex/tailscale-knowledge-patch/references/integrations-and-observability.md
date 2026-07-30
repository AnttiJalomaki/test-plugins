# Integrations and Observability

## Terraform provider

- Provider 0.18 adds `tailscale_logstream_configuration` for Amazon S3 and
  S3-compatible log streaming, and allows `tailscale_tailnet_key` import
  (Tailscale 1.80.0 era).
- The optional `tailscale_acl.reset_acl_on_destroy` property resets the
  tailnet policy file to its default when the resource is destroyed.
- Provider 0.19.0 adds `uploadPeriodMinutes` and `compressionFormat` to
  `tailscale_logstream_configuration` (Tailscale 1.82.0 era).

## Logs and audit

- Network flow logs automatically include information about the logging node
  and the peers it communicates with (since 1.92.1).
- Network flow logs and configuration audit logs can be streamed to Google
  Cloud Storage (since 1.94.1).
- Successful Tailscale SSH authentication on Linux emits a `LOGIN` message to
  the kernel audit subsystem (since 1.94.1).

## Client and Peer Relay metrics

- Clients expose `tailscaled_home_derp_region_id` (since 1.94.1).
- Peer Relay forwarding can be monitored with
  `tailscaled_peer_relay_forwarded_packets_total` and
  `tailscaled_peer_relay_forwarded_bytes_total` (since 1.94.1).
- Peer Relays expose `tailscaled_peer_relay_endpoints` as a user metric (since
  1.96.2).
