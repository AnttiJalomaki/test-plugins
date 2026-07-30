---
name: azure-knowledge-patch
description: Microsoft Azure
version: null
license: MIT
metadata:
  author: Nevaberry
---


# Microsoft Azure Compatibility Guidance

Use this skill when changing Azure infrastructure, deployment automation,
authentication, networking, or service-management code. It is especially
useful for AzureRM 4.x, AzAPI 2.x, recent Azure CLI behavior, Bicep, identity
SDKs, Azure PowerShell, control-plane API migrations, and service retirements.

## How to use this skill

1. Identify the exact client: Terraform provider, CLI, PowerShell module, SDK,
   Bicep CLI, or REST API version.
2. Open the topic reference that matches the resource or workflow being
   changed.
3. Treat explicitly configured values as safer than CLI or service defaults
   for reproducible automation.
4. Recheck scripts that parse JSON or table output; several command shapes and
   field names changed.
5. For a removal or retirement, follow the replacement path before upgrading
   the client or API version.
6. Prefer project manifests, lockfiles, code, tests, and observed service
   behavior when they are more specific than general guidance.

## Reference index

| Reference | Topics |
| --- | --- |
| [Terraform and AzAPI](references/terraform-and-azapi.md) | AzureRM 4.x and AzAPI 2.x upgrades, schemas, state moves, imports, actions, and preflight |
| [Containers and Kubernetes](references/containers-and-kubernetes.md) | AKS, node pools, ACNS, Azure Container Storage, ACR, and Container Apps |
| [Application platform](references/application-platform.md) | App Service, Functions, App Configuration, API Management, and Service Connector |
| [Compute and images](references/compute-and-images.md) | VMs, VMSS, disks, snapshots, galleries, scheduled events, and restore points |
| [Data, storage, and backup](references/data-storage-and-backup.md) | PostgreSQL, MySQL, SQL, Cosmos DB, Storage, Azure Files, NetApp Files, Redis, and Backup |
| [Networking](references/networking.md) | VNets, subnets, gateways, load balancers, public IPs, private endpoints, routing, and WAF |
| [Identity and security](references/identity-and-security.md) | Identity SDKs, sign-in, Entra, Graph, managed identities, RBAC, Key Vault, and MFA |
| [ARM, Bicep, and CLI](references/arm-bicep-and-cli.md) | Deployments, provider registration, API discovery, CLI runtime, packaging, and clouds |
| [Service operations and retirements](references/service-operations-and-retirements.md) | Batch, monitoring, Service Fabric, AI services, HDInsight, IoT, messaging, and retirement inventory |

## Breaking-change triage

Before upgrading automation, check these high-impact boundaries first.

### AzureRM 4.x

- Set `subscription_id` in every provider instance or provide
  `ARM_SUBSCRIPTION_ID`; Azure CLI authentication no longer supplies the active
  subscription implicitly.
- Replace `skip_provider_registration` with
  `resource_provider_registrations`, using `none` for the old skip behavior,
  and add exact namespaces through `resource_providers_to_register`.
- Replace inline subnet `address_prefix` with `address_prefixes`; the inline
  block also supports delegation, policies, routes, and service endpoints.
- Migrate removed `azurerm_sql_*` resources to `azurerm_mssql_*`, MySQL Single
  Server to Flexible Server, and all other removed resource families before
  changing the provider major version.
- Audit renamed `*_enabled` fields, required values, list-to-set changes, and
  defaults for TLS, network SKUs, public access, and upgrade channels.
- Treat newly non-computed service values as possible drift. Configure them or
  use a narrow `lifecycle.ignore_changes` path when Azure owns the value.

```hcl
provider "azurerm" {
  subscription_id                 = var.subscription_id
  resource_provider_registrations = "core"
  resource_providers_to_register  = ["Microsoft.ContainerService"]
  features {}
}
```

### AzAPI 2.x

- Make `body` a native HCL object and consume `output` as an object; remove
  surrounding `jsonencode` and `jsondecode` calls.
- Replace `ignore_body_changes` with a precise Terraform
  `lifecycle.ignore_changes` path.
- Do not use removed provider naming prefixes or suffixes; compose and sanitize
  names in configuration.
- Set `use_msi = true` when managed identity is intended because it no longer
  activates implicitly.
- Remove deprecated retry `multiplier`, `randomization_factor`, and provider
  `maximum_busy_retry_attempts` settings.

```hcl
resource "azapi_resource" "example" {
  type      = "Microsoft.Example/widgets@2025-01-01"
  parent_id = azurerm_resource_group.example.id
  name      = var.name
  body = {
    properties = { enabled = true }
  }
}
```

### Azure CLI automation

- Pin values whose defaults affect provisioning, including VM size, AKS node
  size, PostgreSQL version, MySQL version and IOPS behavior, App Service plan
  operating system and SKU, and Container Apps job scaling.
- AKS cluster creation now omits an SSH key by default. Pass the intended SSH
  configuration rather than depending on the former default.
- `az webapp list-runtimes` returns structured objects and removed the older
  Linux/detail switches; update parsers and filters.
- MySQL backup, restore, geo-restore, and replica workflows no longer accept
  `--storage-redundancy`.
- PostgreSQL Single Server command groups are removed, and Flexible Server
  creation no longer supplies several former database/version defaults.
- The CDN command module is delivered through an extension, so offline and
  controlled installations must make it available explicitly.
- Azure CLI no longer supports Python 3.9 and can ship a newer embedded Python;
  extensions must be compatible with that interpreter.
- Treat all table output as presentation data. Prefer JSON plus a query, and
  still test for renamed, added, or nullable fields.

## Authentication and authorization guardrails

Azure Resource Manager enforces MFA for affected user-identity write
operations. Reads may succeed after a non-MFA sign-in while creates, updates,
or deletes return a claims challenge.

- Prefer managed identities, workload identities, or service principals for
  unattended automation; user service accounts do not bypass MFA.
- Do not use username/password credentials for workflows that can require MFA.
  ROPC cannot satisfy a claims challenge and its public APIs are deprecated.
- Use Azure CLI and Azure PowerShell sign-in flows that understand claims
  challenges when an interactive user must perform a control-plane write.
- Replace Azure AD Graph calls and old manifest shapes with Microsoft Graph.
- Use object IDs in role-assignment operations when avoiding Graph lookup is
  important.
- Constrain `DefaultAzureCredential` with `AZURE_TOKEN_CREDENTIALS=dev`,
  `prod`, or a credential class name when the deployment environment requires
  a deterministic credential chain.
- Allow managed-identity startup enough time for IMDS retry behavior, and do
  not silently accept unsupported user-assigned identity settings.

```bash
AZURE_TOKEN_CREDENTIALS=WorkloadIdentityCredential
az login --identity --client-id "$AZURE_CLIENT_ID"
```

## Key Vault access-model changes

For vault creation through the newer control-plane API, an omitted
`enableRbacAuthorization` selects RBAC. Updating an existing vault does not
silently switch its access model.

- Set `enableRbacAuthorization: false` when a new vault must retain access
  policies.
- Confirm both vault write permission and the ability to create role
  assignments before switching to RBAC, or the operator can lock out access.
- Upgrade management clients before their older control-plane API versions
  retire.
- Check private-endpoint count before adding another endpoint because limits
  are enforced.

## Network and egress changes

- Basic Load Balancer and Basic public IP are retired and unsupported even
  though existing instances can continue operating. Plan a resource-specific
  migration and downtime.
- Keep public IP and load-balancer SKUs compatible during migration. Standard
  public IPs require NSG rules for inbound traffic, and outbound behavior must
  be configured explicitly.
- New virtual networks created through the newer API behavior make subnets
  private by default. Supply NAT Gateway, a Standard load-balancer outbound
  rule, Standard public IP, or firewall/NVA routing when internet egress is
  required.
- Deallocate existing VMs after changing a subnet's default outbound access so
  the setting reaches their NICs.
- A UDR with next hop `Internet` does not itself give a private subnet outbound
  access; service endpoints remain a separate path.

## ARM, Bicep, and API-version workflow

Discover supported API versions and locations from each provider namespace's
`resourceTypes` metadata. Do not assume one version or region applies across a
namespace, and remember that subscription restrictions can narrow the result.

```bash
az provider show --namespace Microsoft.Batch \
  --query "resourceTypes[?resourceType=='batchAccounts'].apiVersions | [0]" \
  --output tsv
```

- Register only the providers the deployment needs. ARM and Bicep register
  providers for explicit template resources, but not necessarily providers
  used only by implicit supporting resources.
- Regional registration can permit creation in one location while the overall
  provider state still reports `Registering`.
- Use Bicep parameter-file `extends` for layered assignments, `@secure()` for
  secret string/object outputs, and direct Bicep `snapshot` for deterministic
  local deployment validation.
- Do not deploy Bicep module identity syntax until backend support is available.

## High-value provider capabilities

- With Terraform 1.8 or later, use AzureRM resource-ID normalization/parsing
  functions and AzAPI resource-ID builders instead of hand-splitting IDs.
- Enable AzAPI preflight to validate resource configuration during planning.
- Use AzAPI provider-identity imports, cross-provider state moves, and bulk
  discovery to avoid unnecessary destroy/recreate workflows.
- Put write-only AzAPI properties in `sensitive_body`, and select sensitive
  response paths separately.
- Use absence-aware reads and stable list-item identity when reconciling
  partially managed collections.

## Retirement checks

Before introducing or extending automation for an older API or service:

1. Query Azure Advisor retirement metadata and impacted-resource
   recommendations.
2. Use Resource Graph's `advisorresources` table to inventory resource IDs,
   retiring features, and dates.
3. Exclude upgrade-only recommendations that do not name a retiring feature.
4. Check sovereign-cloud retirement tooling separately because Advisor
   coverage is incomplete outside public Azure.
5. Replace the retired API Management direct management endpoint with its
   Resource Manager API and migrate developer-portal identity providers from
   ADAL to MSAL with authorization code flow and PKCE.
6. Redesign workflows that depend on SQL control-plane operations with no
   newer stable equivalent; changing only `api-version` is insufficient.

## Final automation checklist

- [ ] Authentication mode and subscription selection are explicit.
- [ ] Provider registration is least-privilege and complete for implicit dependencies.
- [ ] Resource API versions are valid for the type and target location.
- [ ] Removed commands, options, resources, and fields have replacements.
- [ ] Defaults that affect cost, exposure, availability, or placement are pinned.
- [ ] JSON and table output consumers tolerate documented schema changes.
- [ ] Network egress and inbound rules are explicit after SKU or subnet changes.
- [ ] Workload identity replaces user-password automation where MFA can apply.
- [ ] Retirement inventory covers each deployed subscription and cloud.
- [ ] Plan, validation, and representative deployment tests pass before rollout.
