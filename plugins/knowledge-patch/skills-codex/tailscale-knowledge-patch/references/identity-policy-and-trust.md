# Identity, Policy, and Trust

## Device posture

- `ip:country` is generally available for country-based device posture checks
  (since 1.80.0).
- `tsStateEncrypted` can be used when policy depends on client state being
  encrypted at rest (since 1.86.0).

## Grants and routed policy

- Grants and the `via` routing field are generally available as of 1.84.0.
  `via` can require traffic to pass through selected exit nodes, subnet
  routers, or app connectors.
- New tailnets and tailnet policy files that have never been edited use grants
  rather than ACL syntax without changing effective permissions.

## GitOps and visual editing

- The admin console's Policy file management page stores the external URL for
  a GitOps policy repository (since 1.84.0). The older policy-file code comment
  is deprecated; the admin-console value takes precedence when both exist.
- A beta visual policy editor can manage the tailnet policy file (since
  1.86.0).

## Tailnet Lock

- Tailnet Lock is generally available as of 1.84.0. It can require the tailnet
  to verify new node keys supplied by the coordination server before trusting
  them.
- `tailscale lock log --json` returns Authority Update Messages in a stable
  format suitable for parsers (since 1.92.1).
- `tailscale lock status -json` returns tailnet key authority data in a stable
  format (since 1.94.1):

  ```console
  tailscale lock status -json
  ```

## Workload identity federation

- A node can authenticate with workload identity federation by passing
  `--client-id` and `--id-token` to `tailscale up` (since 1.92.1):

  ```console
  tailscale up --client-id=<client-id> --id-token=<identity-token>
  ```

- Federated identities can be managed through the Tailscale API,
  `tailscale-client-go-v2`, and the Terraform provider.
- Workload identities can generate identity tokens automatically, with
  `tailscale up --audience=<audience>` selecting the audience (since 1.94.1).
- GitHub Actions and GitLab CI GitOps integrations support provider-native
  identity-token authentication. Token-exchange failures are shown on the
  admin console's Trust credentials page.
