---
name: tailscale-knowledge-patch
description: Tailscale
version: 1.98.1
license: MIT
metadata:
  author: Nevaberry
---



# Tailscale Knowledge Patch

Use this skill when upgrading or operating Tailscale clients, containers,
Kubernetes Operator resources, Tailscale Services, identity integrations,
tailnet policy, routing, DNS, or observability. Before applying advice, identify
the installed client and component versions, platform, distribution channel,
and whether configuration is interactive, MDM-managed, containerized, or
Operator-managed.

## Reference index

| Reference | Topics |
| --- | --- |
| [release-compatibility.md](references/release-compatibility.md) | Withdrawn and release-candidate builds, rollout regressions, platform support, upgrade caveats |
| [identity-policy-and-security.md](references/identity-policy-and-security.md) | Workload identity, auth and node keys, grants, posture, Tailnet Lock, policy management |
| [networking-services-dns-and-relays.md](references/networking-services-dns-and-relays.md) | Services, Serve and Funnel, exit nodes, subnet routing, DNS, DERP, Peer Relays |
| [kubernetes-operator.md](references/kubernetes-operator.md) | ProxyGroups, multi-cluster routing, API proxy, recording, audit, Ingress, Recorder, multi-tailnet controls |
| [containers-ci-terraform-and-logging.md](references/containers-ci-terraform-and-logging.md) | Container behavior, CI authentication and caching, Terraform resources, external log storage |
| [client-platforms-and-managed-policies.md](references/client-platforms-and-managed-policies.md) | Windows, macOS, iOS, tvOS, Android, Linux, QNAP, OpenWrt, MDM policies |
| [cli-ssh-taildrive-and-observability.md](references/cli-ssh-taildrive-and-observability.md) | CLI automation, Tailscale SSH, Taildrive, metrics, flow-log fields, stable JSON output |

## Avoid unsafe release selections

- Do not deploy Linux v1.98.1: that build was withdrawn because of a MagicDNS
  interaction regression. Treat v1.98.0 as a release candidate, not the prior
  stable fallback.
- Do not standardize on v1.86.0. Its rollout was halted after regressions.
  Standalone macOS deployments using `EncryptState` need v1.86.4 for the
  fresh-install startup-crash fix; v1.86.2 fixes the state-file read problem
  that could trigger device re-approval.
- Treat v1.90.0, v1.92.0, v1.94.0, v1.96.0, v1.96.1, and v1.98.0 as
  release-candidate builds intended for testing. Use the corresponding stable
  line where appropriate.
- macOS 12 is the minimum supported macOS version in current guidance.
- Certificate-serial allowlists for the Windows binary must include the new
  signing-certificate serial used by v1.84.2. Subject and issuer matching alone
  will not reveal this rotation.

## Update deprecated or changed configuration

- Replace the macOS/iOS `ForceEnabled` policy with `AlwaysOn.Enabled` and, when
  users may override the enforced connection, `AlwaysOn.OverrideWithReason`.
- Replace macOS `TailscaleOnboardingSeen` with `OnboardingFlow`. Use
  `AppIntroShown` for the distinct first-login Welcome modal.
- Remove `TS_EXPERIMENTAL_KUBE_API_EVENTS` from Operator deployments and grant
  Kubernetes API event access through tailnet ACLs.
- Expect Tailscale Services to advertise at startup. Set
  `TS_EXPERIMENTAL_SERVICE_AUTO_ADVERTISEMENT=false` only when automatic
  advertisement is unwanted.
- On macOS, stop scripting `tailscale drive`; sharing Taildrive directories
  moved to the client GUI.
- New or never-edited policy files use grants syntax. Existing effective
  permissions do not change merely because the default authoring syntax did.
  Prefer grants for new policy and use `via` when traffic must traverse chosen
  exit nodes, subnet routers, or app connectors.

## Harden CLI automation

- Do not repeat the same CLI flag. Duplicate flags are rejected. Container
  image v1.84.2 is required if `TS_EXTRA_ARGS` must set `--accept-dns` under
  the stricter parser.
- Some significant CLI actions now ask for `y/n` confirmation. Automation must
  anticipate prompts and must not assume an affected command remains
  noninteractive.
- Use machine-readable interfaces where available:

  ```console
  tailscale dns status --json
  tailscale lock log --json
  tailscale lock status -json
  ```

- Gate dependent processes with `tailscale wait`, and verify an expected node
  address with `tailscale ip --assert=<specific-ip-address>`.
- Use the literal `release-candidate` track with `tailscale version --track`
  and `tailscale update --track`; do not invent an abbreviated track name.

## Choose the current authentication path

For direct workload identity federation, supply the client ID and externally
issued identity token:

```console
tailscale up --client-id=<client-id> --id-token=<identity-token>
```

Workload identities can also obtain tokens automatically. Select the token
audience with:

```console
tailscale up --audience=<audience>
```

The container image supports OAuth and workload identity federation. GitHub
Actions and GitLab CI GitOps integrations support provider-native identity
tokens. The Kubernetes Operator can also use provider-native tokens; when its
Helm values omit an OAuth client secret, use an Operator version containing the
no-secret rendering fix.

For `tsrecorder`, keep an auth key in a mounted secret and point to it rather
than copying it into the environment:

```console
export TS_AUTHKEY_FILE=/run/secrets/tailscale-auth-key
```

Remember that the `AuthKey` system policy applies only while no user is logged
in. Node-key sealing is enabled by default on Linux, Windows, and macOS, and
existing Linux nodes migrate automatically when upgraded.

## Configure Services, routing, and DNS

- Tailscale Services are generally available and can be hosted by `tsnet`
  nodes. Clients accept Service virtual IPs independently of `--accept-routes`,
  Operator egress proxies can target those VIPs, and a Service destination may
  be remote.
- Serve and Funnel can prepend a PROXY protocol header so the destination sees
  the original source address and port.
- Use `auto:any` for an exit node that tracks the current recommendation:

  ```console
  tailscale set --exit-node=auto:any
  ```

  On managed Windows or macOS devices, combine `ExitNodeID=auto:any` with
  `ExitNode.AllowOverride` when exit-node use is mandatory but user selection
  is allowed.
- Exit-node use no longer prevents sending all domains to admin-configured DNS
  resolvers. Tailnet-only resolvers can use DNS-over-TCP fallback, and Linux
  MagicDNS can work with plain `resolv.conf` even without a DNS manager.
- Give Peer Relays fixed endpoints with
  `tailscale set --relay-server-static-endpoints`; EC2-hosted relays can also
  advertise addresses discovered from instance metadata.
- On Linux subnet routers and exit nodes, act on the IP-forwarding health check.
  Current firewall setup also marks routed traffic to avoid reverse-path
  filtering drops.

## Apply Kubernetes Operator behavior deliberately

- Use a `ProxyGroup` for highly available, active-active Ingress or Tailscale
  Kubernetes Service proxies. A group can multiplex applications and support
  cross-cluster failover based on cluster-wide `EndpointSlice` health.
- Use `ProxyGroup` type `kube-apiserver` for a highly available Kubernetes API
  proxy. Recording supports `kubectl exec`, `attach`, and `debug`; beta audit
  logging can supplement or replace full recordings.
- Multiple Recorder replicas require S3. The default Recorder is a
  single-replica `StatefulSet` with filesystem storage.
- Use the `tailscale.com/http-redirect` Ingress annotation for HTTP-to-HTTPS
  redirects. An omitted managed-Ingress path defaults to `/`.
- Use the `Tailnet` custom resource for multi-tailnet access and
  `ProxyGroupPolicy` to control which namespaces may create ProxyGroups.
- Treat tag and service uniqueness validation errors as configuration errors:
  ACL tags must be valid, and only one Tailscale Kubernetes Service in a
  cluster may reference a given Tailscale Service.

## Manage endpoint policy by platform

- `AlwaysOn.Enabled` on Windows connects at sign-in and stays active without
  the GUI, including on headless hosts. Installation also starts the GUI for
  every signed-in user.
- `ReconnectAfter` limits permitted disconnect time on Windows, Android, and
  macOS. `tailscale down --reason` supplies the explanation associated with a
  managed disconnect.
- `EncryptState` is TPM-backed on Windows and Keychain-backed on macOS; the
  App Store macOS client always uses Keychain. Linux uses
  `tailscaled --encrypt-state` for TPM-backed state.
- Use `Hostname` to override the OS-reported device hostname, and
  `EnableDNSRegistration` on Windows to control Active Directory DNS
  registration.
- Check the platform-specific policy reference before reusing a key across
  platforms: `UseSystemProxy`, `advertiseExitNode`, `AuthBrowser.macos`,
  `HideDockIcon`, `OnboardingFlow`, and `AppIntroShown` are macOS-specific in
  this guidance.

## Verify security and observability

- Tailnet Lock can require verification of new coordination-server node keys.
  Prefer its stable JSON log and status interfaces when building parsers.
- `tsStateEncrypted` reports whether client state is encrypted at rest.
  `ip:country` is available for country-based device posture checks.
- Successful Tailscale SSH authentication on Linux emits a kernel audit
  `LOGIN` record. Direct-IP SSH works without MagicDNS, and current servers
  accept clients that start authentication with `publickey` rather than
  sending `none` first.
- Flow logs include details for the logging node and its peers. Monitor client
  home DERP selection and Peer Relay forwarded packets, bytes, and endpoint
  count with the metrics listed in the observability reference.
