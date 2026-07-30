# Plugins, Agents, and Delivery

## External and official plugins

- Enterprise can execute plugins externally. Place an already extracted
  artifact in the plugin directory before registration; do not expect Vault to
  extract it.
- Community Edition registration also supports an extracted artifact
  directory.
- Vault can download official auth and secrets plugins from
  `releases.hashicorp.com` as a beta. Enterprise CLI registration exposes this
  through `-download`.
- Detailed API-client registration variants return the registration response
  alongside an error. `RegisterPlugin` and `RegisterPluginWithContext` are
  deprecated.
- Enterprise can override pinned versions when creating or updating database
  engines and when enabling or tuning auth and secrets backends.
- The UI recognizes and updates first-party external plugins and can mount a
  registered external plugin at a selected version.
- Plugin list responses include a SHA-256 sum.

## Enterprise signing-key compatibility

Vault Enterprise 1.19.17, 1.20.11, 1.21.6, and 2.0.1 cannot register
Enterprise plugins released on or after April 21, 2026. Their renewed signing
key fails verification, although existing registrations keep working. Upgrade
to 1.19.18, 1.20.12, 1.21.7, or 2.0.2 or later in the matching line.

## Container distribution

- Images run as the `vault` user by default from 1.19.16.
- Images are exported as compressed OCI image layouts.
- UBI variants use UBI 10 minimal.
- The 1.19.17 image required runtime `IPC_LOCK`; 1.19.18 removed built-in
  `cap_ipc_lock`, so containers cannot call `mlock()`. Set
  `disable_mlock = true` and prevent swapping externally.
- The 1.19.16 image has a distinct unresolved `setfcap` startup failure with an
  available workaround.

## Agent, Proxy, and Kubernetes delivery

- Supported auto-auth methods can reauthenticate when credentials change with
  `enable_reauth_on_new_credentials`; certificate auto-auth watches its cert
  and key files.
- Vault Agent can perform Enterprise External CA ACME workflows. Templates
  re-render after certificate issuance or renewal.
- Vault Agent's built-in API proxy is deprecated and pending removal. Use
  Vault Proxy for proxy behavior.
- Vault Secrets Operator delivers protected secrets to application pods as
  CSI-backed shared volumes without native Kubernetes Secret objects.

## SDK Docker test clusters

SDK Docker helpers use `github.com/moby/moby` instead of
`github.com/docker/docker`. `DockerClusterNode.UpdateConfig` accepts complete
cluster options and supports seals, KMS libraries, and entropy augmentation.

## Enterprise licensing

Vault Enterprise accepts IBM PAO license keys. Configure the required
`license_entitlement` stanza when using that license type.
