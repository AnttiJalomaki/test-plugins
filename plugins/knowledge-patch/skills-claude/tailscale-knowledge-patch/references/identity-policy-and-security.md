# Identity, Policy, and Security

## Authenticate workloads and automation

Nodes can authenticate through workload identity federation by passing an
external identity token and its client ID (1.92.1):

```console
tailscale up --client-id=<client-id> --id-token=<identity-token>
```

Federated identities can be managed with the Tailscale API,
`tailscale-client-go-v2`, and the Terraform provider. Workload identities can
also generate identity tokens automatically; use `--audience` to select the
token audience (1.94.1):

```console
tailscale up --audience=<audience>
```

GitHub Actions and GitLab CI GitOps integrations accept provider-native
identity-token authentication. Token-exchange failures are reported on the
admin console's Trust credentials page (1.94.1). The container and Kubernetes
Operator variants have their own support and configuration considerations;
see their topic references.

On iOS and tvOS, auth keys work with custom coordination servers. Apple TV can
use an auth key to join a tailnet. A custom coordination server URL may use
HTTP with an explicit custom port (1.80.0).

The `AuthKey` system policy applies only while no user is logged in (1.96.2).
Do not expect it to replace or reauthenticate an active user session.

## Protect node keys and state

Node-key renewal preserves existing connections while the client
reauthenticates (1.90.1). Node-key sealing is generally available and enabled
by default on Linux, Windows, and macOS; upgrading an existing Linux node
automatically migrates it to sealed node keys (1.90.1).

Tailnet Lock is generally available and can require the tailnet to verify new
node keys received from the coordination server before trusting them (1.84.0).
For programmatic consumers, `tailscale lock log --json` returns Authority
Update Messages in a stable format (1.92.1), and
`tailscale lock status -json` returns stable tailnet key-authority data
(1.94.1).

Encrypted-state controls differ by platform (1.86.0):

- Linux supports TPM-backed storage with `tailscaled --encrypt-state`.
- Windows provides the TPM-backed `EncryptState` system policy.
- macOS uses `EncryptState` to store state in Keychain; the App Store client
  always uses Keychain. Standalone macOS v1.86.4 applies policy changes without
  restarting the system extension.
- The `tsStateEncrypted` posture attribute reports whether client state is
  encrypted at rest.

## Author tailnet policy

Grants and the `via` routing field are generally available (1.84.0). New
tailnets and tailnet policy files that have never been edited use grants rather
than ACL syntax, without changing effective permissions. Use `via` to require
traffic to traverse selected exit nodes, subnet routers, or app connectors.

App connectors are generally available for securing access from a tailnet to
SaaS applications (1.84.0). A beta visual policy editor can manage the tailnet
policy file (1.86.0).

The admin console's Policy file management page stores the external URL for a
GitOps policy repository. The older policy-file code comment is deprecated;
if both values exist, the admin-console setting wins (1.84.0).

## Build posture checks

The `ip:country` geolocation attribute is generally available for device
posture checks (1.80.0). `tsStateEncrypted` can enforce encrypted-at-rest
client state (1.86.0). The open-source macOS variant reports `node:osVersion`
for OS-version posture evaluation (1.96.2).

## Apply security corrections

- Use a stable-line build containing the web-interface CSRF correction when
  login attempts unexpectedly fail (1.86.0).
- Control-plane connections through a CONNECT HTTPS proxy must verify the
  destination hostname; the v1.86 stable line restores this behavior after a
  regression (1.86.0).
- Successful Tailscale SSH authentication on Linux emits a `LOGIN` message to
  the kernel audit subsystem, enabling host audit-policy correlation (1.94.1).
