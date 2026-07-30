# CLI, SSH, Taildrive, and Observability

## Write robust CLI automation

`tailscale configure` and its subcommands are no longer alpha, except for
`tailscale configure kubeconfig` (1.80.0). The Standalone macOS system-extension
and Linux systray subcommands are described in the platform reference.

CLI commands reject multiple occurrences of the same flag (1.84.0). Audit
scripts that append flags through more than one configuration layer. Container
image v1.84.2 specifically restores `TS_EXTRA_ARGS` support for setting
`--accept-dns` under this parser.

`tailscale down` accepts `--reason`, supporting managed policies that require
an explanation for disconnecting (1.84.0).

Significant CLI actions can ask for `y/n` confirmation, making previously
noninteractive invocations interactive (1.88.1). Automation should explicitly
handle each affected command's prompt behavior.

Use `tailscale wait [flags]` to wait until Tailscale resources are ready for
binding. Use `tailscale ip --assert=<specific-ip-address>` to fail unless the
argument matches one of the node's Tailscale addresses (1.96.2):

```console
tailscale wait
tailscale ip --assert=100.64.0.1
```

The `release-candidate` track works with both version checks and updates
(1.96.2):

```console
tailscale version --track=release-candidate
tailscale update --track=release-candidate
```

For DNS automation, both `tailscale dns query` and `tailscale dns status`
accept `--json` (1.96.2).

## Integrate stable Tailnet Lock output

`tailscale lock log --json` returns Authority Update Messages in a stable
format (1.92.1). `tailscale lock status -json` returns stable tailnet
key-authority data (1.94.1):

```console
tailscale lock log --json
tailscale lock status -json
```

Retain each command's documented spelling: the log form uses `--json`, while
the status example uses `-json`.

## Operate Tailscale SSH

Linux, macOS, and FreeBSD servers accept SSH clients that skip the `none`
authentication method and begin with `publickey`. This behavior was restored
in v1.80.2 to match v1.78.x and earlier (1.80.0).

Tailscale SSH works when the destination is given as an IP address and
MagicDNS is disabled (1.88.1). On Linux, every successful Tailscale SSH
authentication emits a `LOGIN` record to the kernel audit subsystem (1.94.1).

## Share files with Taildrive

Taildrive folder sharing works on Linux and other Unix-like hosts that lack the
`su` command, and shared files remain consistently accessible (1.88.1).

The macOS client no longer exposes `tailscale drive`; configure Taildrive
directory sharing in the client GUI (1.90.1).

## Consume metrics and flow logs

Clients export `tailscaled_home_derp_region_id`, which identifies the home DERP
region (1.94.1).

Peer Relays export cumulative forwarding metrics
`tailscaled_peer_relay_forwarded_packets_total` and
`tailscaled_peer_relay_forwarded_bytes_total` (1.94.1). They also export the
`tailscaled_peer_relay_endpoints` user-metric gauge for advertised endpoint
count (1.96.2).

Network flow logs include node details for the node producing the log and the
peers with which it communicates (1.92.1). Network flow and configuration
audit logs can be streamed to Google Cloud Storage (1.94.1), while S3 and
S3-compatible streaming is covered in the Terraform and logging reference.
