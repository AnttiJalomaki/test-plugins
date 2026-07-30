# Application Platform

Use this reference for App Service, Functions, App Configuration, API Management, Service Connector, and application deployment.

## ADAL-based developer-portal identity providers are retired

_Source batch `api-management-retirements`._

On September 30, 2025, API Management's provided developer portal stopped
supporting ADAL-based Microsoft Entra ID and Azure AD B2C identity providers;
without migration, user sign-in and sign-up stop working even though the API
Management service remains available. Change the application's redirect URI
to the single-page application platform, select `MSAL` as the identity
provider's client library, update the configuration, and republish the
developer portal; the replacement uses authorization code flow with PKCE.

## Anonymous App Configuration access

_Azure CLI `2.83.0`._

App Configuration commands now accept `anonymous` for `--auth-mode`.

## API Management backends

_Azure CLI `2.85.0`._

The new `az apim backend` command group supports API Management backend
services.

## App Configuration custom token audiences

_Azure CLI `2.70.0`._

`az appconfig` operations using `--auth-mode login` can now use a custom token
audience.

## App Configuration feature-management schema and timestamps

_Azure CLI `2.69.0`._

Key-value import/export and feature show/list commands now understand the
Microsoft feature-management schema. File exports can set
`AZURE_APPCONFIG_FM_COMPATIBILE` for backward compatibility, and datetime
inputs for key-value restore/show/list and revision list accept timezone
offsets.

## App Configuration network security perimeters

_Azure CLI `2.87.0`._

App Configuration store create/update operations and the
`network-security-perimeter-configuration` command now support Network
Security Perimeter configuration.

## App Configuration retention and feature tag filters

_Azure CLI `2.76.0`._

`az appconfig create/update` can set the key-value revision retention period.
Feature `list`, `delete`, and `set` operations now support tag filters.

## App Configuration serialization

_Azure CLI `2.78.0`._

`az appconfig kv export` escapes keys only for properties-file output.
`az appconfig kv set` and `import` now accept JSON comments.

## App Configuration snapshot references

_Azure CLI `2.87.0`._

`az appconfig kv set-snapshot-reference` creates a snapshot-reference
key-value, and `az appconfig kv list` can list key-values from a snapshot
reference.

## App Configuration tag filters and dry runs

_Azure CLI `2.75.0`._

Tag filtering is now supported by App Configuration key-value
export/import/list/delete, restore, and revision-list operations.
`az appconfig kv import`, `export`, and `restore` also accept `--dry-run`.

## App Configuration telemetry

_Azure CLI `2.85.0`._

App Configuration store create/update can link an Application Insights
resource, and `az appconfig feature set` can enable telemetry for a feature
flag.

## App Service asynchronous scaling

_Azure CLI `2.78.0`._

`az appservice plan create` and `update` accept `--async-scaling-enabled`.

## App Service command changes

_Azure CLI `2.69.0`._

`az functionapp deployment slot create` gains `--https-only`. On Linux,
`az webapp list-runtimes` no longer returns JBoss `_byol` entries, so scripts
that select those runtime identifiers must be updated.

## App Service deployment diagnostics and conversion

_Azure CLI `2.86.0`._

`az webapp up` and `az webapp deploy` accept `--enriched-errors` to return
detailed deployment-failure logs. `az webapp sitecontainers convert` can now
convert Docker Compose multi-container apps to Sitecontainers mode.

## App Service domain-label scope and container conversion

_Azure CLI `2.76.0`._

`az webapp create` accepts `--domain-name-scope` for DNL scope selection, and
`az webapp sitecontainers convert` switches an app between sitecontainers and
classic configuration.

## App Service hostname scope and release channels

_Azure CLI `2.85.0`._

`az logicapp create` and `az webapp up` accept `--domain-name-scope` to
select the uniqueness scope for the default hostname. `az webapp update`
accepts `--platform-release-channel` to set the app's platform release
channel.

## App Service lifecycle and zoning

_Azure CLI `2.73.0`._

App Service Environment create/update/delete no longer supports ASEv2.
`az functionapp plan update` can now update zone redundancy for Flex plans.

## App Service output and runtime discovery

_Azure CLI `2.84.0`._

`az webapp config access-restriction show` now always returns values in camel
case, so consumers of its output must use that casing. `az webapp list
runtimes` no longer relies on hardcoded runtime lists and includes previously
missing Java versions.

## App Service plan operating-system default

_Azure CLI `2.88.0`._

Without an explicit `--hyper-v`, `az appservice plan create` now defaults to
Linux. Pass `--is-linux false` when creating a Windows App Service plan.

## App Service VNet routing behavior

_Azure CLI `2.84.0`._

For API version `2024-11-01`, Web App create/configuration and Web App or
Function App VNet-integration commands now use the site-level outbound VNet
routing property.

## Consumption-to-Flex function migration

_Azure CLI `2.77.0`._

The new `az functionapp flex-migration` command group supports migrating CV1
function apps to Flex.

## Deployment-slot VNet inheritance

_Azure CLI `2.71.0`._

`az webapp deployment slot create` now gives a new slot the same VNet
integration settings as its source slot, matching Portal-created slots.

## Flex Consumption certificates

_Azure CLI `2.88.0`._

`az functionapp config ssl` now supports site-scoped certificates for Flex
Consumption. `az functionapp flex-migration` can also migrate Linux
Consumption apps that have certificates.

## Function App update strategies

_Azure CLI `2.87.0`._

`az functionapp update-strategy config set` sets or updates a Function App's
update-strategy configuration, while `config show` retrieves it.

## Linux App Service plan default

_Azure CLI `2.86.0`._

When `--sku` is omitted for a Linux web app, `az appservice plan create` now
defaults to `P0V3`. The command also recognizes the `PREMIUM0V3` tier for
elastic scale, so automation that depends on another plan size must pass it
explicitly.

## Linux Web App site containers and Kudu warm-up

_Azure CLI `2.70.0`._

Linux web apps gain the `az webapp sitecontainers` command group. Deployment
through `az webapp up`, `az webapp deploy`, or
`az webapp deployment source config-zip` can use `--enable-kudu-warmup` to
warm Kudu before deploying.

## Managed-instance-aware App Service locations

_Azure CLI `2.82.0`._

`az appservice list-locations` accepts `--managed-instance-enabled` when
discovering locations that support managed instances.

## Neon Postgres Service Connector

_Azure CLI `2.71.0`._

Service Connector's workload-specific `connection create neon-postgres`
commands can create connections to Neon Postgres Serverless.

## New App Configuration and App Service SKUs

_Azure CLI `2.72.0`._

`az appconfig create` and `az appconfig update` support the Developer SKU.
`az appservice plan create` supports the Pv4 and Pmv4 App Service Plan
families.

## Service Connector identity and Fabric targeting

_Azure CLI `2.70.0`._

`az containerapp connection create redis` accepts `--system-identity`.
`az webapp connection create fabric-sql` gains `--fabric-workspace-uuid` and
`--fabric-sql-db-uuid` for selecting the Fabric workspace and SQL database.

## Site-scoped Web App certificates

_Azure CLI `2.87.0`._

`az webapp create --site-scoped-certs` controls whether site-scoped
certificates are enabled for a new app.

## Structured Web App runtime discovery

_Azure CLI `2.87.0`._

In a breaking output change, `az webapp list-runtimes` now returns objects
with `os`, `runtime`, `version`, `config`, `support`, and `end_of_life`
fields instead of a flat string list. Use the new `--runtime` and `--support`
filters; `--linux` and `--show-runtime-details` have been removed.

## The direct management API is retired

_Source batch `api-management-retirements`._

The optional direct management API, disabled by default but previously
available in the Premium, Standard, Basic, and Developer tiers, retired on
March 15, 2025. Replace tools that call
`https://<service-name>.management.azure-api.net` with equivalent operations
in the standard Azure Resource Manager-based API Management REST API, and
disable the direct API if it is still enabled; the API Management instance
itself is otherwise unaffected.

## Web App transport encryption

_Azure CLI `2.84.0`._

`az webapp create` and `az webapp update` accept
`--end-to-end-encryption-enabled` for encryption between the front end and
workers. Creation also accepts `--min-tls-version` and
`--min-tls-cipher-suite`.

## Web App worker-count validation

_Azure CLI `2.74.0`._

`az webapp config set` no longer performs CLI validation of the number of
workers, so that check no longer rejects the request before it reaches Azure.

## Zone-redundant Elastic Premium Functions

_Azure CLI `2.80.0`._

`az functionapp plan create` now supports zone redundancy for Elastic Premium
SKUs.
