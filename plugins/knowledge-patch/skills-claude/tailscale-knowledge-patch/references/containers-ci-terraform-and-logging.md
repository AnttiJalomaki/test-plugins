# Containers, CI, Terraform, and Log Storage

## Configure the container image

Container image v1.80 can load `TS_SERVE_CONFIG` even when HTTPS is disabled
for the tailnet, provided that the supplied Serve configuration declares no
HTTPS endpoint (1.80.0).

CLI duplicate flags are rejected beginning with v1.84.0. That parser change
initially blocked `TS_EXTRA_ARGS` from setting `--accept-dns`; image v1.84.2
restores the container use case (1.84.0).

Container image v1.92.3 restores `iptables` operation on hosts without
`nftables` support (1.92.1). Container image v1.94.1 supports both OAuth and
workload identity federation authentication (1.94.1).

Services advertise automatically at startup. To retain manual advertisement,
set (1.96.2):

```text
TS_EXPERIMENTAL_SERVICE_AUTO_ADVERTISEMENT=false
```

When the image starts in Kubernetes, v1.86.2 clears Pod-specific state and
uses external node IP addresses as static endpoints to improve direct
connectivity to `ProxyGroup` Pods (1.86.0).

## Run in CI systems

The Tailscale GitHub Action is generally available on macOS and Windows
runners. Cache downloaded Tailscale binaries by supplying the string input
(1.82.0):

```yaml
use-cache: 'true'
```

GitHub Actions and GitLab CI GitOps integrations support authentication with
provider-native identity tokens (1.94.1). Token-exchange failures appear on
the admin console's Trust credentials page.

## Manage resources with Terraform

Terraform provider v0.18 adds the following behavior (1.80.0):

- `tailscale_logstream_configuration` manages streaming to Amazon S3 and
  S3-compatible services.
- `tailscale_tailnet_key` supports import.
- Optional `tailscale_acl.reset_acl_on_destroy` resets the tailnet policy file
  to its default when the ACL resource is destroyed. Enable it only when that
  destructive lifecycle behavior is intended.

Provider v0.19.0 adds `uploadPeriodMinutes` and `compressionFormat` to
`tailscale_logstream_configuration` (1.82.0). Federated identities can also be
managed through the Terraform provider (1.92.1).

## Stream and interpret logs

Network flow logs and configuration audit logs can stream to Google Cloud
Storage (1.94.1). For S3 or S3-compatible destinations, use the Terraform
log-stream resource and its upload-period and compression controls.

Network flow logs automatically include information about the logging node
and the peers with which it communicates (1.92.1). Consumers should use these
fields rather than performing a separate join solely to recover endpoint node
identity.
