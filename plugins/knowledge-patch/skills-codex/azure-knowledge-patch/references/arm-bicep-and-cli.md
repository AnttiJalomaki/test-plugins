# Arm Bicep And Cli

Use this reference for ARM and Bicep deployment, provider registration, Azure CLI runtime and packaging, clouds, and core output behavior.

## ARM and Bicep auto-registration has an explicit-resource boundary

_Source batch `arm-api-versions-and-registration`._

Portal resource creation typically registers its provider, and ARM template or
Bicep deployments automatically register providers for resource types defined
in the template. Providers needed only by implicit supporting resources, such
as monitoring or security integrations not present in the template, must be
registered separately.

## ARM export format and deployment validation

_Azure CLI `2.76.0`._

`az group export --export-format` selects the exported template format.
Deployment `create`, `validate`, and `what-if` expose `--validation-level` at
every scope.

## Azure Linux 2.0 packaging support removed

_Azure CLI `2.75.0`._

Azure CLI packages no longer support Azure Linux (Mariner) 2.0; installations
on that release must move to a supported platform.

## Bicep overwrite and deployment-stack controls

_Azure CLI `2.84.0`._

`az bicep decompile-params --force` can overwrite existing output files.
Deployment-stack create and validate commands at resource-group,
subscription, and management-group scope gain `--validation-level` and
`--resources-without-delete-support`; stack deletion also gains the latter
option for resources that cannot be deleted when no longer managed.

## Bleu known-cloud support

_Azure CLI `2.85.0`._

Bleu is now included in the CLI's Known Clouds list.

## CDN moves out of the core CLI

_Azure CLI `2.88.0`._

The entire CDN module is now supplied through `azure-cli-extensions`, so
automation and offline installations that use CDN commands must make the
extension available.

## CLI platform support

_Azure CLI `2.73.0`._

Azure CLI packages on RHEL and CentOS Stream now use Python 3.12, and Ubuntu
20.04 is no longer supported.

## Consumption usage null values

_Azure CLI `2.75.0`._

`az consumption usage list` now emits a JSON null for missing values instead
of the literal string `None`, which changes parsing for affected output.

## Custom-cloud endpoint controls

_Azure CLI `2.75.0`._

`az cloud register` and `az cloud update` accept
`--endpoint-microsoft-graph-resource-id` for the Microsoft Graph endpoint and
`--skip-endpoint-discovery` to suppress automatic endpoint discovery.

## Custom-cloud endpoint discovery

_Azure CLI `2.73.0`._

With `az cloud register` or `update --endpoint-resource-manager`, endpoint
discovery now finds data-plane endpoints automatically and no longer returns
a `gallery` endpoint. Consumers of the discovered endpoint set must tolerate
that field being absent.

## Deployment what-if diagnostics

_Azure CLI `2.75.0`._

The pretty-printed result from `az deployment what-if` now includes potential
changes, warnings, and diagnostic messages.

## Discover API versions and locations from provider metadata

_Source batch `arm-api-versions-and-registration`._

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

## Extendable parameter files

_Source batch `bicep-language-and-cli`._

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

## Interactive expression console

_Source batch `bicep-language-and-cli`._

Bicep CLI 0.42.1 adds `bicep console`, a REPL for expressions, variables,
multi-line input, user-defined types and functions, and `load*()` calls
resolved from the launch directory. It also accepts piped or redirected input,
but has no Azure-context functions such as `resourceGroup()`, session
persistence, completions, or `az bicep` equivalent.

```bash
bicep console
echo "parseCidr('10.144.0.0/20')" | bicep console
```

## Least-privilege provider registration

_Source batch `arm-api-versions-and-registration`._

Registration requires the provider's `/register/action` permission, which
Contributor and Owner include, and it can add an application for the provider
to the Microsoft Entra tenant, typically through the Windows Azure Service
Management API. Register only providers that are ready for use; a provider
cannot be unregistered while its resource types still exist in the
subscription.

## Local deterministic deployment snapshots

_Source batch `bicep-language-and-cli`._

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

## Module identity is not yet deployable

_Source batch `bicep-language-and-cli`._

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

## Python 3.13 packaging

_Azure CLI `2.77.0`._

Azure CLI now supports Python 3.13, and packaged builds embed Python 3.13.7.

## Python 3.14 packaging

_Azure CLI `2.88.0`._

Azure CLI now supports Python 3.14 and ships Python 3.14.5 as its embedded
runtime; extensions that depend on the embedded interpreter must be
compatible with that version.

## Python 3.9 support removed

_Azure CLI `2.80.0`._

Azure CLI no longer supports Python 3.9.

## Regional registration completion and new locations

_Source batch `arm-api-versions-and-registration`._

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

## Resource-list table output

_Azure CLI `2.79.0`._

`az resource list --output table` now includes `provisioningState`, changing
the table schema for consumers that parse its columns.

## Secure outputs

_Source batch `bicep-language-and-cli`._

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

## Version-pinned Bicep installation

_Azure CLI `2.81.0`._

`az bicep install --version` now installs the requested version without
requiring `bicep.use_binary_from_path` to be explicitly set to `false`;
previously the installation could be skipped without that setting.
