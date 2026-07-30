---
name: azure-knowledge-patch
description: Microsoft Azure
version: null
license: MIT
metadata:
  author: Nevaberry
---


# Microsoft Azure Knowledge Patch

Use this skill when writing, reviewing, or upgrading Azure infrastructure,
automation, authentication, SDK, CLI, PowerShell, Terraform, ARM, or Bicep
work. Start with the migration hazards below, then open the reference whose
topic matches the resource or tool being changed.

Treat project manifests, provider lockfiles, deployed resource state, command
help, API metadata, and tests as authoritative. Azure behavior often depends
on cloud, region, resource API version, provider version, CLI extension, and
whether a resource is new or already exists.

## Reference index

| Reference | Topics |
| --- | --- |
| [terraform-providers.md](references/terraform-providers.md) | AzureRM 4, AzAPI 2.x, provider setup, HCL bodies and outputs, imports, actions, schemas, retries |
| [identity-authentication-and-graph.md](references/identity-authentication-and-graph.md) | Credential chains, Azure CLI and PowerShell sign-in, claims challenges, MFA, managed identity, Microsoft Graph |
| [aks-and-containers.md](references/aks-and-containers.md) | AKS, node pools, Azure Container Registry, Container Apps, container instances, IoT |
| [compute-and-app-platform.md](references/compute-and-app-platform.md) | VMs, VMSS, disks, images, Backup, App Service, Functions, Batch, AI, HDInsight, Service Fabric |
| [networking.md](references/networking.md) | Private subnet defaults, retired SKUs, VNets, gateways, routing, load balancing, WAF, private endpoints |
| [data-and-storage.md](references/data-and-storage.md) | PostgreSQL, MySQL, SQL, Cosmos DB, App Configuration, Storage, Azure Files, NetApp Files, Key Vault |
| [deployment-governance-and-cli.md](references/deployment-governance-and-cli.md) | ARM, Bicep, provider registration, deployments, generic CLI behavior, platform support, retirement inventory |

## Breaking migrations first

### AzureRM 4

- Set `subscription_id` in every provider instance or provide
  `ARM_SUBSCRIPTION_ID`; Azure CLI authentication no longer supplies the
  active subscription implicitly.
- Replace `skip_provider_registration` with
  `resource_provider_registrations`; add exact namespaces through
  `resource_providers_to_register`.
- Audit removed `azurerm_sql_*`, MariaDB, MySQL Single Server, media, lab,
  analytics, integration-environment, and other retired resources before
  changing the provider constraint.
- Apply exact field migrations, especially AKS `*_enabled` names, diagnostic
  setting `enabled_log`, positive network-policy booleans, and singular
  Container App Job blocks.
- Remove positional assumptions where AzureRM changed lists to sets.
- Make service-owned computed values explicit or narrowly ignore them when a
  refresh exposes upgrade-time drift.

### AzAPI 2.x

- Express `body` as native HCL and read `output` as an HCL object. Remove
  surrounding `jsonencode` and `jsondecode`.
- Replace `ignore_body_changes` with a precise
  `lifecycle.ignore_changes` path.
- Opt into managed identity with `use_msi = true`; it is no longer selected
  implicitly.
- Expect default computed output to refresh. Use response projections or
  `disable_default_output` when full output is unnecessary.
- Remove deprecated retry multiplier, randomization, and provider busy-retry
  controls; current provider defaults supersede the early 2.0 example.
- Use `sensitive_body` and sensitive response projections instead of placing
  secrets in ordinary state-visible body or output values.

### Azure CLI automation

- Pin the CLI and required extensions in reproducible automation, and check
  the installed command's help before relying on a newly added argument.
- Pass important defaults explicitly. VM size, AKS node size and SSH-key
  behavior, App Service plan OS/SKU, MySQL and PostgreSQL versions, storage
  redundancy, and container settings have all changed.
- Parse JSON properties by name. Table columns, property casing, null
  representation, and entire output shapes can change.
- Treat confirmation prompts as breaking for unattended scripts; pass the
  command's explicit confirmation option only after validating the target.
- Remove deprecated or deleted commands and options rather than suppressing
  their warnings. Some command groups have moved to extensions.
- Account for the embedded Python runtime when an extension carries native or
  version-sensitive dependencies.

## Authentication and identity

- Do not use username/password authentication for users. ROPC and
  `UsernamePasswordCredential` cannot satisfy mandatory MFA.
- Prefer managed identity, workload identity, or a service principal for
  unattended control-plane writes.
- A control-plane write made with a user identity may return a claims
  challenge even when reads succeed. Use clients that can surface and satisfy
  the challenge.
- Constrain `DefaultAzureCredential` with `AZURE_TOKEN_CREDENTIALS` when a
  deterministic developer or production chain is required.
- Managed identity selected as the only default credential skips the IMDS
  availability probe and can retry for roughly 70 seconds; size startup
  deadlines accordingly.
- Use explicit `--client-id`, `--object-id`, or `--resource-id` for
  user-assigned managed-identity login.
- Select a subscription directly in supported CLI credential options when
  ambient CLI state is unsafe.
- Migrate Azure AD Graph requests and legacy manifest shapes to Microsoft
  Graph. Do not treat old extended access as a continuing fallback.
- In Az.Accounts 5, handle `Get-AzAccessToken` tokens as `SecureString`.

## Network and access defaults

- New virtual networks created with the newer API default subnets to no
  implicit outbound access. Configure NAT Gateway, a Standard Load Balancer
  outbound rule, a Standard public IP, or firewall/NVA routing explicitly.
- Stop and deallocate existing VMs after changing a subnet's
  `defaultOutboundAccess` so the setting reaches their NICs.
- Do not expect an Internet-next-hop UDR alone to give a private subnet
  outbound connectivity.
- Migrate Basic Load Balancers and Basic public IPs through the owning
  resource's supported path. Standard public load balancing also needs
  explicit NSG and outbound design.
- Preserve public IPs by making them static before disassociation during
  migration.
- Match load-balancer and public-IP SKU and tier; mixed-SKU configurations are
  not an in-place bridge.
- Treat private-endpoint provider support as command-version specific and
  verify the target resource provider before scripting approval workflows.

## Resource providers, ARM, and Bicep

- Register only providers that are ready for use and grant the narrow
  `/register/action` permission when possible.
- A provider can remain globally `Registering` while a completed region is
  ready. Re-run registration when a provider later adds a needed region.
- Template and Bicep deployments auto-register only providers for explicit
  resource types; register providers needed solely by implicit integrations.
- Discover API versions and locations from each provider resource type's
  metadata. Provider support does not override subscription restrictions.
- Use `@secure()` on string or object outputs that must stay out of deployment
  history and logs.
- Treat Bicep module identity syntax as nondeployable until backend support is
  available.
- Use direct `bicep snapshot` for deterministic local deployment snapshots;
  it is not an `az bicep` command.

## Service-specific safety checks

### AKS and containers

- Validate removed preview-only AzureRM AKS fields before provider upgrades;
  use AzAPI only with clear state ownership boundaries.
- Pass AKS node VM size explicitly when capacity and cost must be stable.
- Treat `--no-ssh-key` as the current create default and opt into the intended
  SSH posture explicitly.
- Test node-pool upgrade controls, rollback, disruption budgets, and
  unavailable-node limits against a nonproduction pool first.
- Keep container-storage generation and enable/disable syntax aligned with the
  installed CLI.

### Compute and App Service

- Pass VM or VMSS size and security type explicitly when request stability
  matters.
- Re-test scripts that parse disk, snapshot, gallery, runtime, resiliency, or
  scheduled-event output.
- Select the App Service plan operating system explicitly; omitted values can
  now produce Linux plans.
- Verify VNet routing, end-to-end encryption, certificate scope, and runtime
  discovery when updating Web Apps or Function Apps.
- Treat backup restore, vault reconfiguration, disk creation during attach,
  and zone movement as stateful operations requiring target validation.

### Data, storage, and Key Vault

- Pin PostgreSQL and MySQL versions, network mode, HA/zonal resiliency, storage
  type, and redundancy instead of inheriting CLI defaults.
- Remove PostgreSQL Single Server commands and unsupported server versions
  from automation.
- Remove MySQL `--storage-redundancy` from backup, restore, geo-restore, and
  replica scripts where the command no longer accepts it.
- Treat storage TLS 1.0 and 1.1 inputs as TLS 1.2; do not present the older
  values as enforceable configuration.
- For new Key Vaults on the current stable control-plane API, omitted
  `enableRbacAuthorization` means RBAC. Set it to `false` explicitly only
  when retaining access policies is intentional.
- Ensure the operator can create role assignments before switching a vault's
  access model, or the change can lock out administration.

## Retirement handling

- Inventory retirements through Advisor and Resource Graph, but account for
  incomplete service/resource coverage and sovereign-cloud differences.
- Replace API Management's retired direct management API with its ARM-based
  management API.
- Migrate developer-portal identity providers from ADAL to MSAL authorization
  code flow with PKCE and republish the portal.
- Move SQL control-plane callers away from `2014-04-01`; some old operation
  groups have no direct stable equivalent and require workflow redesign.
- Treat an Azure resource that remains operational after retirement as
  unsupported, not as evidence that migration can be deferred safely.

## Validation checklist

1. Identify the cloud, subscription, region, API version, provider/CLI
   version, and extension set used by the actual deployment.
2. Inspect state and live resource defaults before changing configuration.
3. Search the references for the exact resource type, command, option, or
   output property being changed.
4. Run plan, what-if, validation, or read-only discovery before mutation.
5. Exercise authentication with the same identity type and claims policy used
   in production.
6. Test output parsers against captured JSON from the installed tool.
7. Confirm rollback and recovery paths for network, identity, data, and
   control-plane migrations.
