# Deployment, governance, and CLI lifecycle

ARM and Bicep authoring, resource-provider registration, deployment behavior, generic CLI changes, platform support, and retirement inventory.

## ARM, resource providers, and Bicep

### ARM and Bicep auto-registration has an explicit-resource boundary (arm-api-versions-and-registration)

Portal resource creation typically registers its provider, and ARM template or
Bicep deployments automatically register providers for resource types defined
in the template. Providers needed only by implicit supporting resources, such
as monitoring or security integrations not present in the template, must be
registered separately.

### Discover API versions and locations from provider metadata (arm-api-versions-and-registration)

Query a provider's `resourceTypes` metadata instead of assuming that every
resource type uses the same API versions or regions. Returned locations
describe provider support, but subscription restrictions can still make a
listed region unavailable.

```bash
az provider show --namespace Microsoft.Batch \
  --query "resourceTypes[?resourceType=='batchAccounts'].apiVersions | [0]" \
  --output tsv
az provider show --namespace Microsoft.Batch \
  --query "resourceTypes[?resourceType=='batchAccounts'].locations | [0]" \
  --output tsv
```

### Extendable parameter files (bicep-language-and-cli)

Bicep CLI 0.44.1 adds `extends` and `base` for layering parameter
assignments. A derived file can extend one base, chains are allowed, and
base/intermediate files use `using none`; a derived assignment replaces the
inherited value unless an object or array is explicitly merged with spreads
from `base`.

```bicep
// base.bicepparam
using none
param location = 'westus'
param tags = {
  owner: 'platform'
  environment: 'dev'
}

// prod.bicepparam
using './main.bicep'
extends './base.bicepparam'
param tags = {
  ...base.tags
  environment: 'prod'
}
```

Only parameter assignments are inherited. Variables, user-defined types, and
imported functions in a base file aren't exposed to derived files.

### Interactive expression console (bicep-language-and-cli)

Bicep CLI 0.42.1 adds `bicep console`, a REPL for expressions, variables,
multi-line input, user-defined types and functions, and `load*()` calls
resolved from the launch directory. It also accepts piped or redirected input,
but has no Azure-context functions such as `resourceGroup()`, session
persistence, completions, or `az bicep` equivalent.

```bash
bicep console
echo "parseCidr('10.144.0.0/20')" | bicep console
```

### Least-privilege provider registration (arm-api-versions-and-registration)

Registration requires the provider's `/register/action` permission, which
Contributor and Owner include, and it can add an application for the provider
to the Microsoft Entra tenant, typically through the Windows Azure Service
Management API. Register only providers that are ready for use; a provider
cannot be unregistered while its resource types still exist in the
subscription.

### Local deterministic deployment snapshots (bicep-language-and-cli)

Bicep CLI 0.41.2 adds `snapshot` for producing a normalized representation
from a `.bicepparam` file and validating later code against it without
deploying or consulting live Azure state. Use direct `bicep`, not `az bicep`;
offline evaluation can be supplied with `--subscription-id`,
`--resource-group`, `--location`, `--tenant-id`, and `--management-group`
context when environment functions need concrete values.

```bash
bicep snapshot --mode overwrite main.bicepparam
bicep snapshot --mode validate main.bicepparam
```

### Module identity is not yet deployable (bicep-language-and-cli)

Bicep 0.36.1 recognizes a user-assigned managed identity on a module, intended
to let the module access services such as Key Vault. Backend services don't
yet support the capability, so code using this syntax can't currently rely on
it at deployment time.

```bicep
param identityId string

module workload './workload.bicep' = {
  name: 'workload'
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${identityId}': {}
    }
  }
}
```

### Regional registration completion and new locations (arm-api-versions-and-registration)

Registration runs separately for every supported region. Do not block resource
creation merely because the provider remains in `Registering`; creation can
proceed in a target region once registration has completed there; run the
registration operation again when a provider adds a location that the
subscription needs.

```bash
az provider register --namespace Microsoft.Batch
az provider show --namespace Microsoft.Batch \
  --query registrationState --output tsv
```

### Secure outputs (bicep-language-and-cli)

Since Bicep 0.35.1, `@secure()` can mark string or object outputs, including
outputs returned through modules. ARM then omits their values from deployment
history, portal views, logs, and command-line output; wrap arrays or numbers
in an object or serialize them to a string when they must remain secret.

```bicep
@secure()
param suppliedToken string

@secure()
output returnedToken string = suppliedToken
```

## Deployment and resource inspection

### ARM export format and deployment validation (2.76.0)

`az group export --export-format` selects the exported template format.
Deployment `create`, `validate`, and `what-if` expose `--validation-level` at
every scope.

### Bicep overwrite and deployment-stack controls (2.84.0)

`az bicep decompile-params --force` can overwrite existing output files.
Deployment-stack create and validate commands at resource-group,
subscription, and management-group scope gain `--validation-level` and
`--resources-without-delete-support`; stack deletion also gains the latter
option for resources that cannot be deleted when no longer managed.

### Deployment what-if diagnostics (2.75.0)

The pretty-printed result from `az deployment what-if` now includes potential
changes, warnings, and diagnostic messages.

### Resource-list table output (2.79.0)

`az resource list --output table` now includes `provisioningState`, changing
the table schema for consumers that parse its columns.

### Version-pinned Bicep installation (2.81.0)

`az bicep install --version` now installs the requested version without
requiring `bicep.use_binary_from_path` to be explicitly set to `false`;
previously the installation could be skipped without that setting.

## CLI behavior, clouds, and platform support

### API Management backends (2.85.0)

The new `az apim backend` command group supports API Management backend
services.

### Azure Linux 2.0 packaging support removed (2.75.0)

Azure CLI packages no longer support Azure Linux (Mariner) 2.0; installations
on that release must move to a supported platform.

### Bleu known-cloud support (2.85.0)

Bleu is now included in the CLI's Known Clouds list.

### CDN moves out of the core CLI (2.88.0)

The entire CDN module is now supplied through `azure-cli-extensions`, so
automation and offline installations that use CDN commands must make the
extension available.

### Classic-administrator option removed (2.73.0)

`az role assignment list` no longer accepts
`--include-classic-administrators`.

### CLI platform support (2.73.0)

Azure CLI packages on RHEL and CentOS Stream now use Python 3.12, and Ubuntu
20.04 is no longer supported.

### Consumption usage null values (2.75.0)

`az consumption usage list` now emits a JSON null for missing values instead
of the literal string `None`, which changes parsing for affected output.

### Custom-cloud endpoint controls (2.75.0)

`az cloud register` and `az cloud update` accept
`--endpoint-microsoft-graph-resource-id` for the Microsoft Graph endpoint and
`--skip-endpoint-discovery` to suppress automatic endpoint discovery.

### Custom-cloud endpoint discovery (2.73.0)

With `az cloud register` or `update --endpoint-resource-manager`, endpoint
discovery now finds data-plane endpoints automatically and no longer returns
a `gallery` endpoint. Consumers of the discovered endpoint set must tolerate
that field being absent.

### Preview macOS installation methods (2.85.0)

Azure CLI adds additional preview installation methods on macOS.

### Python 3.13 packaging (2.77.0)

Azure CLI now supports Python 3.13, and packaged builds embed Python 3.13.7.

### Python 3.14 packaging (2.88.0)

Azure CLI now supports Python 3.14 and ships Python 3.14.5 as its embedded
runtime; extensions that depend on the embedded interpreter must be
compatible with that version.

### Python 3.9 support removed (2.80.0)

Azure CLI no longer supports Python 3.9.

### RHEL 10 and CentOS Stream 10 packages (2.76.0)

Azure CLI packaging now supports RHEL 10 and CentOS Stream 10.

## Retirements and inventory

### ADAL-based developer-portal identity providers are retired (api-management-retirements)

On September 30, 2025, API Management's provided developer portal stopped
supporting ADAL-based Microsoft Entra ID and Azure AD B2C identity providers;
without migration, user sign-in and sign-up stop working even though the API
Management service remains available. Change the application's redirect URI
to the single-page application platform, select `MSAL` as the identity
provider's client library, update the configuration, and republish the
developer portal; the replacement uses authorization code flow with PKCE.

### Advisor retirement metadata and impacted-resource APIs (service-retirement-calendar)

Azure Advisor classifies upgrade and retirement recommendations under API
category `HighAvailability` and subcategory `ServiceUpgradeAndRetirement`.
Use the provider-level metadata endpoint to list recommendation metadata and
the subscription endpoint to list recommendations with impacted resources;
`recommendationControl` is a legacy filter property planned for deprecation.

```http
GET https://management.azure.com/providers/Microsoft.Advisor/metadata?api-version=2025-01-01&$filter=recommendationCategory%20eq%20'HighAvailability'%20and%20recommendationSubCategory%20eq%20'ServiceUpgradeAndRetirement'&$expand=ibiza

GET https://management.azure.com/subscriptions/<subscription-id>/providers/Microsoft.Advisor/recommendations?api-version=2025-01-01&$filter=Category%20eq%20'HighAvailability'%20and%20SubCategory%20eq%20'ServiceUpgradeAndRetirement'&$expand=ibiza,details
```

The expanded responses include links, recommendation details, and recommended
actions. Advisor retirement recommendations currently cover only public Azure,
and both service coverage and impacted-resource coverage are incomplete;
sovereign and national-partner clouds require the Azure Retirement Impact
Analyzer.

### Resource Graph retirement inventory (service-retirement-calendar)

The `advisorresources` table exposes the affected resource ID, retiring
feature, and retirement date. Upgrade-only recommendations share this
subcategory but have no retiring feature, so filter them out when building a
retirement inventory.

```kusto
advisorresources
| where type == "microsoft.advisor/recommendations"
| where properties.category == "HighAvailability"
| where properties.extendedProperties.recommendationSubCategory == "ServiceUpgradeAndRetirement"
| extend retirementFeatureName = properties.extendedProperties.retirementFeatureName
| extend retirementDate = properties.extendedProperties.retirementDate
| extend resourceId = properties.resourceMetadata.resourceId
| where retirementFeatureName != ''
| project retirementFeatureName, retirementDate, resourceId
```

### SQL Database 2014-04-01 control-plane retirement (service-retirement-calendar)

Azure SQL Database control-plane API version `2014-04-01` now retires on
June 30, 2027, rather than June 30, 2026. The deadline covers every operation,
including servers, databases, elastic pools, managed instances, and related
SQL resources; the primary stable migration target is `2021-11-01`.

| `2014-04-01` operation group | `2021-11-01` replacement |
| --- | --- |
| Database table auditing policies | Database blob auditing policies |
| Database threat detection policies | Database advanced threat protection settings |
| Disaster recovery configurations | Failover groups |
| Extensions | Database extensions |
| Restorable dropped databases | Restorable dropped managed databases |
| Service objectives | Capabilities |
| Transparent data encryption activities/configurations | Transparent data encryptions |

Database connection policies, elastic-pool activities, elastic-pool database
activities, queries, query statistics, query texts, recommended elastic pools,
and service-tier advisors have no newer stable equivalent; workflows using
them must be redesigned rather than only changing the `api-version`.

### The direct management API is retired (api-management-retirements)

The optional direct management API, disabled by default but previously
available in the Premium, Standard, Basic, and Developer tiers, retired on
March 15, 2025. Replace tools that call
`https://<service-name>.management.azure-api.net` with equivalent operations
in the standard Azure Resource Manager-based API Management REST API, and
disable the direct API if it is still enabled; the API Management instance
itself is otherwise unaffected.
