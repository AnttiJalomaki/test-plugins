# Service Operations And Retirements

Use this reference for Batch, monitoring, Service Fabric, AI services, HDInsight, IoT, messaging, resource retirement discovery, and remaining service operations.

## Action-group incident receivers and identities

_Azure CLI `2.74.0`._

`az monitor action-group` now supports `--incident-receivers`,
`--mi-user-assigned`, and `--mi-system-assigned`.

## Advisor retirement metadata and impacted-resource APIs

_Source batch `service-retirement-calendar`._

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

## AI Foundry command groups

_Azure CLI `2.80.0`._

The `az cognitiveservices account connection`,
`az cognitiveservices account project`, and
`az cognitiveservices account project connection` groups manage AI Foundry
resources, and `az cognitiveservice agent` is a new command group.

## Azure CNI static block allocation

_Azure CLI `2.75.0`._

`az aks create` and `az aks nodepool add` accept
`--pod-ip-allocation-mode` to configure Azure CNI Static Block Allocation.

## Azure US Government Graph endpoint

_Azure CLI `2.76.0`._

The `AZURE_US_GOV_CLOUD` profile now sets `active_directory_graph_resource_id`
to `https://graph.microsoftazure.us/`.

## Base64 key-operation digests

_Azure CLI `2.68.0`._

`az keyvault key sign` and `az keyvault key verify` now accept a base64-encoded
string in `--digest`, so callers no longer need a different representation
for that input.

## Batch pool argument removals

_Azure CLI `2.80.0`._

`az batch pool create` no longer accepts `--target-communication` or
`--resource-tags`; pool `reset` and `set` also drop
`--target-communication`.

## Classic-administrator option removed

_Azure CLI `2.73.0`._

`az role assignment list` no longer accepts
`--include-classic-administrators`.

## Cognitive Services project management and kind changes

_Azure CLI `2.78.0`._

`az cognitiveservices account create` accepts `--allow-project-management`,
and `account update --kind` supports OpenAI-to-AIServices kind changes and back.

## Cross-tenant user-delegation SAS

_Azure CLI `2.86.0`._

The blob, container, share, file, queue, and filesystem `generate-sas`
commands accept `--user-delegation-tid` to issue a user-delegation SAS for a
different tenant.

## Expanded Batch task and JSON configuration

_Azure CLI `2.69.0`._

Batch job creation gains `--job-manager-task-application-package-references`
and `--on-all-tasks-complete`. Job-schedule creation gains that application
package option plus `--job-metadata` and
`--job-manager-task-environment-settings`; schedule set/reset gains
`--job-max-task-retry-count` and `--job-max-wall-clock-time`.

`--json-file` is now accepted by job disable, node reboot, node scheduling
disable, and pool autoscale evaluate. Pool creation gains
`--start-task-environment-settings` and `--start-task-max-task-retry-count`,
while pool reset gains `--start-task-resource-files` and
`--target-node-communication-mode`.

## Flexible Consumption location discovery

_Azure CLI `2.71.0`._

`az functionapp list-flexconsumption-locations` accepts `--details` to return
more location information and `--runtime` to select a runtime.

## Grafana-backed dashboards

_Azure CLI `2.82.0`._

`az monitor dashboard` now supports dashboards with Grafana.

## HDInsight credential operations

_Azure CLI `2.79.0`._

`az hdinsight credentials show` retrieves current cluster credentials, and
`az hdinsight credentials update` changes them.

## Hosted AI Foundry agents

_Azure CLI `2.82.0`._

`az cognitiveservices agent create` can create and deploy a hosted agent in
AI Foundry.

## Hosted-agent log streaming

_Azure CLI `2.83.0`._

`az cognitiveservices agent logs show` streams console logs for hosted agents.
Agent `create` and `start` accept `--show-logs`, and `start` also accepts
`--timeout`.

## IoT Hub device streams move to an extension

_Azure CLI `2.77.0`._

The `az iot hub devicestream` command group is now supplied by the
`azure-iot` extension rather than Azure CLI itself.

## IoT Hub minimum TLS version

_Azure CLI `2.70.0`._

`az iot hub update` accepts `--min-tls-version` to change the hub's minimum
TLS version.

## Linux container startup logs

_Azure CLI `2.87.0`._

The new `az webapp log startup` commands list and display Linux container
startup logs.

## Premium-ingress replica arguments removed

_Azure CLI `2.80.0`._

`az containerapp env premium-ingress` operations no longer accept
`--min-replicas` or `--max-replicas`.

## Preview macOS installation methods

_Azure CLI `2.85.0`._

Azure CLI adds additional preview installation methods on macOS.

## Removed Batch commands and options

_Azure CLI `2.69.0`._

The deprecated `az batch certificate create/list/show/delete`,
`az batch node reimage`, and `az batch node remote-desktop` commands are
removed. Batch pool creation also removes `--application-licenses`,
`--certificate-references`, `--os-family`, and `--os-version`; pool set/reset
removes `--certificate-references`.

## Resource Graph retirement inventory

_Source batch `service-retirement-calendar`._

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

## RHEL 10 and CentOS Stream 10 packages

_Azure CLI `2.76.0`._

Azure CLI packaging now supports RHEL 10 and CentOS Stream 10.

## Service Fabric cluster names from parameter files

_Azure CLI `2.77.0`._

When a parameters file supplies `cluster_name`, `az sf cluster create` now
uses that value.

## Service Fabric managed-cluster controls

_Azure CLI `2.76.0`._

Managed-cluster network security rules accept `--source-addr-prefix`,
`--dest-addr-prefix`, `--source-port-range`, and `--dest-port-range`.
`az sf managed-node-type update` can also change `--vm-size` and `--tags`.

## Service Fabric update argument removals

_Azure CLI `2.80.0`._

`az sf managed-application update` drops `--service-type-policy`,
`--upgrade-replica-set-check-timeout`, `--max-porcent-unhealthy-partitions`,
`--max-porcent-unhealthy-replicas`, `--max-porcent-unhealthy-services`, and
`--max-porcent-unhealthy-apps`. `az sf application update` drops
`--service-type-policy`, `--upgrade-replica-set-check-timeout`,
`--instance-close-duration`, `--consider-warning-as-error`,
`--max-percent-unhealthy-partitions`, `--max-percent-unhealthy-replicas`, and
`--max-percent-unhealthy-deployed-applications`.

## Shared-image deletion guard

_Azure CLI `2.71.0`._

`az sig image-version create` and `az sig image-version update` accept
`--block-deletion-before-end-of-life` to prevent deletion before the image
version's end-of-life date.

## Spot placement command replacement

_Azure CLI `2.76.0`._

Use `az compute-recommender spot-placement-score` in place of
`az compute-recommender spot-placement-recommender`.

## SQL Database 2014-04-01 control-plane retirement

_Source batch `service-retirement-calendar`._

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

## TLS 1.0 and 1.1 inputs are coerced to TLS 1.2

_Azure CLI `2.83.0`._

On storage account create/update, passing `--min-tls-version tls1_0` or
`tls1_1` now sets the value to `tls1_2`.
