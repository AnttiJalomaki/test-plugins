---
name: tailscale-knowledge-patch
description: Tailscale
version: 1.98.1
license: MIT
metadata:
  author: Nevaberry
---


# Tailscale Knowledge Patch

Use this skill when implementing, reviewing, or operating current Tailscale
clients, policy, containers, Kubernetes resources, Services, relays, or
automation. It is especially useful when older examples may hide a changed
default, a retired setting, an interactive prompt, or a stable-release
boundary.

## How to apply this patch

1. Identify the deployed client, container, Operator, Terraform provider, and
   platform independently; their release numbers and availability can differ.
2. Check release caveats before copying commands into unattended automation.
3. Check platform policy names and defaults before producing MDM profiles.
4. For Kubernetes, distinguish the Operator version from the image used by
   proxy, Recorder, and DNSConfig workloads.
5. Prefer the stable JSON interfaces documented here when writing parsers.
6. Follow the topic reference for complete behavior and compatibility notes.

## Reference index

| Reference | Topics |
| --- | --- |
| [CLI and release automation](references/cli-and-release-automation.md) | Configuration commands, CI, noninteractive use, JSON output, release tracks, stable-line caveats |
| [Managed clients and platforms](references/managed-clients-and-platforms.md) | System policy, authentication, encrypted state, desktop and mobile behavior, platform support |
| [Kubernetes and containers](references/kubernetes-and-containers.md) | Operator proxies, API proxy, Recorder, DNSConfig, Ingress, container authentication and networking |
| [Networking, routing, and Services](references/networking-routing-and-services.md) | DERP, DNS, exit nodes, Serve, Funnel, Services, Peer Relay, subnet routing |
| [Identity, policy, and trust](references/identity-policy-and-trust.md) | Grants, posture, GitOps, workload identity, Tailnet Lock, policy editing |
| [Integrations and observability](references/integrations-and-observability.md) | Terraform resources, log streaming, metrics, flow logs, audit events |

## Breaking changes and deprecations

### Treat release candidates and withdrawn builds explicitly

- Do not infer stability from an even version number. Several `.0` builds were
  release candidates or internal-only; use the stable-line guidance in the
  release reference.
- Do not roll out Linux 1.98.1: that build was withdrawn because of a MagicDNS
  interaction regression.
- The 1.86.0 rollout was halted. On affected macOS installations, use the
  later stable-line fixes for state-file reads and `EncryptState` startup.

### Make CLI automation deterministic

- A command rejects repeated occurrences of the same flag. Normalize generated
  argument lists before invocation.
- Significant CLI actions can prompt for `y/n`. Do not assume an affected
  command remains noninteractive merely because an older invocation was.
- Container 1.84.0 could not apply `--accept-dns` through `TS_EXTRA_ARGS`;
  use image 1.84.2 or later when relying on that combination.

### Replace retired controls

- On macOS and iOS, use `AlwaysOn.Enabled` and
  `AlwaysOn.OverrideWithReason`; `ForceEnabled` is deprecated.
- On macOS, use `OnboardingFlow`; `TailscaleOnboardingSeen` is deprecated.
- Store a GitOps policy repository URL in the admin console. The policy-file
  code comment is deprecated, and the admin-console value wins if both exist.
- On macOS, share Taildrive directories through the GUI; `tailscale drive` is
  no longer provided there.
- Do not set `TS_EXPERIMENTAL_KUBE_API_EVENTS`; configure Kubernetes API event
  behavior through Tailscale ACLs.

### Account for changed defaults

- New or never-edited tailnet policy files use grants syntax, with unchanged
  effective permissions.
- Node-key sealing is enabled by default on Linux, Windows, and macOS.
- Services advertise automatically at startup. Set
  `TS_EXPERIMENTAL_SERVICE_AUTO_ADVERTISEMENT=false` to opt out.
- `Recorder` defaults to a one-replica `StatefulSet` with filesystem storage.
- `AuthKey` system policy applies only while no user is logged in.
- macOS 12 is the minimum supported macOS release.

## High-value CLI patterns

### Machine-readable DNS state

```console
tailscale dns status --json
tailscale dns query example.internal --json
```

Both DNS commands support JSON output. Select fields deliberately rather than
parsing the human-readable form.

### Wait for resources and assert an address

```console
tailscale wait
tailscale ip --assert=100.64.0.1
```

`tailscale wait [flags]` waits until Tailscale resources are available for
binding. `tailscale ip --assert` succeeds only when the requested address is
one of the node's Tailscale IP addresses.

### Track the recommended exit node

```console
tailscale up --exit-node=auto:any
tailscale set --exit-node=auto:any
```

On Linux, Windows, and macOS, `auto:any` follows the recommended exit node and
can switch as node availability or network conditions change. Supported Apple
and Windows clients also expose this as the Recommended picker choice.

### Use stable Tailnet Lock JSON

```console
tailscale lock log --json
tailscale lock status -json
```

Use these stable formats for Authority Update Messages and tailnet key
authority status instead of scraping display output.

### Authenticate a workload

Use an externally supplied identity token:

```console
tailscale up --client-id=<client-id> --id-token=<identity-token>
```

Or ask a workload identity to generate a token for an audience:

```console
tailscale up --audience=<audience>
```

Keep workload identity, OAuth, and auth-key flows distinct in deployment
configuration. Containers and the Kubernetes Operator have their own support
and version constraints.

## High-value policy and security behavior

### Grants and routed access

Grants and the `via` field are generally available. Use `via` when traffic
must traverse selected exit nodes, subnet routers, or app connectors. Validate
generated policy against grants semantics instead of assuming ACL-only syntax.

### Encrypted state

- Linux supports TPM-backed state with `tailscaled --encrypt-state`.
- Windows uses the TPM-backed `EncryptState` policy.
- macOS uses `EncryptState` to put state in Keychain; the App Store client
  always uses Keychain.
- Test posture with `tsStateEncrypted` when access depends on encryption at
  rest.

### Managed disconnection and exit-node choice

- `tailscale down --reason` records a reason.
- `ReconnectAfter` limits how long users may remain disconnected on Windows,
  Android, and macOS where supported.
- Combine `ExitNode.AllowOverride` with `ExitNodeID=auto:any` on Windows or
  macOS to require an exit node while permitting user selection.

## High-value Services and routing behavior

### Tailscale Services

Tailscale Services are generally available and can be hosted by `tsnet`.
Clients accept Service virtual IPs on every platform regardless of
`--accept-routes`; Operator egress proxies can target those VIPs. A Service
destination may also be remote.

### Preserve client addresses through Serve or Funnel

Serve and Funnel can prepend a PROXY protocol header so the destination sees
the original client source IP and port. Enable it only when the destination
expects PROXY protocol.

### Linux routers

Linux reports a health check when IP forwarding is wrong for a subnet router
or exit node. Its firewall setup also uses `src_valid_mark` with `connmark` so
reverse-path filtering does not discard routed packets.

## High-value Kubernetes behavior

### Highly available application proxies

Operator-managed Ingresses and Tailscale Kubernetes Services can share a
`ProxyGroup` with multiple active replicas, multiplex applications, and expose
backends across clusters. Cluster-wide `EndpointSlice` watching allows
failover when one cluster has no healthy backend.

### Kubernetes API proxy

Use a `ProxyGroup` of type `kube-apiserver` for a highly available API server
proxy. Recording covers `kubectl exec`, `kubectl attach`, and `kubectl debug`;
audit events can be enabled alongside or instead of full recordings.

### Recorder storage

Multiple Recorder replicas require S3 storage. A generated Recorder service
account can use AWS IRSA, avoiding static S3 credentials. Verify the current
default before relying on an omitted replica or storage field.

### Multi-tailnet and namespace controls

Use the `Tailnet` custom resource for multi-tailnet access and
`ProxyGroupPolicy` to control which namespaces may create ProxyGroups.
Ingress and egress ProxyGroup pods can request a replacement auth key when
needed.

## Verification checklist

- Confirm the intended build was actually published for the target platform.
- Confirm commands cannot stop on an unexpected confirmation prompt.
- Confirm generated arguments do not repeat flags.
- Confirm MDM payloads use current policy names and platform scopes.
- Confirm service advertisement is intentionally automatic or explicitly
  disabled.
- Confirm Operator authentication works without an accidentally required OAuth
  secret.
- Confirm multi-replica Recorders use S3.
- Confirm custom DERP certificate and firewall pinning data are current.
- Confirm parsers consume stable JSON output where one is available.
