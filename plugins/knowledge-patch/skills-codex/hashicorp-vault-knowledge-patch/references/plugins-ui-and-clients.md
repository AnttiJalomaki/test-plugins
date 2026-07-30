# Plugins, UI, clients, and integrations

Use this reference for plugin packaging and registration, API/SDK migration, GUI route and workflow changes, Terraform behavior, and operator integrations.

## External and extracted plugins

- Enterprise supports running external plugins, but the operator must place an extracted artifact in the plugin directory before registration; Vault does not extract it on demand.
- Community Edition registration also accepts an extracted artifact directory.
- Confirm filesystem ownership and the artifact digest before registration.

## Official plugin downloads

- Vault can automatically download official auth and secrets plugins from `releases.hashicorp.com` as a beta capability (since 1.20).
- Enterprise exposes the flow through `vault plugin register -download`.
- Treat network access, provenance, and beta behavior as explicit deployment dependencies.

## Registration API migration

- Detailed API-client registration methods return the registration response together with an error.
- `RegisterPlugin` and `RegisterPluginWithContext` are deprecated. Move callers that need server response detail to the detailed variants.
- Enterprise can override pinned versions when creating or updating database engines and when enabling or tuning auth and secrets backends (since 2.0).
- Plugin-list responses include a SHA-256 sum.
- The UI recognizes and updates first-party external plugins and can mount a registered external plugin at a selected version.

## Signing-key compatibility

Enterprise 1.19.17, 1.20.11, 1.21.6, and 2.0.1 cannot register Enterprise plugins released on or after April 21, 2026. Use 1.19.18, 1.20.12, 1.21.7, or 2.0.2 or later in the corresponding line. Existing registrations are unaffected.

## SDK Docker helpers

- SDK Docker test helpers use `github.com/moby/moby` instead of `github.com/docker/docker` (since 1.20).
- `DockerClusterNode.UpdateConfig` accepts the full cluster options and supports seals, KMS libraries, and entropy augmentation.
- Update imports and test fixtures together; partial migration can leave incompatible option types.

## Web login configuration

- Enterprise can configure default and backup auth methods shown on the login form (since 1.20).
- `/vault/auth?with=` now identifies only an auth mount path and renders a simplified form.
- Selecting another method no longer rewrites the `with` parameter. Do not treat it as a general auth-method selector.
- The UI supports TLS certificate-auth login from 2.0.

## Namespace workflows

- The Enterprise namespace picker can search, filter, and navigate without reauthentication.
- Enterprise 2.0 adds guided namespace creation through a questionnaire, after which setup can continue through the UI, CLI, or Terraform.
- Keep CLI or API procedures available for the known EGP interaction that can block root-token GUI access to `sys/internal/ui/mounts` in a child namespace.

## Secrets-engine routes and pagination

- UI URLs move from `/secrets` to `/secrets-engines` in 2.0. Update bookmarks, tests, and route-aware automation.
- The secrets-engine list no longer supports bulk deletion.
- In 1.21 and 2.0, changing **Items per page** while away from page 1 can show an empty or partial secrets-engine table. Return to page 1 before changing the size, or refresh and retry there (`upgrade-safety`).

## Identity federation screens

- Enterprise GUI workflows configure workload identity federation for AWS, Azure, and GCP integrations.
- Secret Sync destination screens support workload identity federation for those providers from 2.0.

## Community TOTP screen

The Community Edition UI can list and add TOTP accounts, keep codes hidden until requested, and display expiry timers (since 1.20).

## ACL policy editor

The 2.0 GUI includes a visual editor that produces ACL policy snippets. Review generated paths and capabilities before applying them; visual generation does not replace policy testing.

## Terraform provider

The Enterprise Vault provider supports Terraform ephemeral resources and write-only attributes for KV and database secrets (since 1.20). Ensure state and plan handling preserve the intended non-persistence properties.

## Vault Secrets Operator

Vault Secrets Operator can map protected secrets directly into application pods through CSI-backed shared volumes, avoiding native Kubernetes Secret objects (since 1.21). Validate node-plugin access, pod lifecycle, and volume permissions.

## Vault Agent proxy retirement

Built-in API proxy support in Vault Agent is deprecated and pending removal. Migrate applications that require proxy behavior to Vault Proxy (`upgrade-safety`). This does not remove Agent template or ACME workflows.

## UI-safe operating practice

1. Update saved URLs and browser tests for `/secrets-engines`.
2. Resolve pagination anomalies from page 1 before concluding mounts are missing.
3. Use CLI or API fallbacks for EGP-blocked namespace access and unsupported bulk actions.
4. Verify selected plugin version, pin override, SHA-256 sum, and extracted artifact.
5. Keep beta download and GUI-generated policy flows behind review and validation.
