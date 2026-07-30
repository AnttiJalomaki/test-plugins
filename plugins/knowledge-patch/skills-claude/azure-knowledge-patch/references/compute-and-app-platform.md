# Compute and application platform

Virtual machines, scale sets, disks, images, backup, App Service, Functions, Batch, AI, ARO, HDInsight, and Service Fabric.

## Virtual machines and scale sets

### Automatic VM application upgrades (2.81.0)

`az vm application set` and `az vmss application set` accept
`--enable-automatic-upgrade` to enable automatic application upgrades.

### Auxiliary tokens during VM creation (2.73.0)

`az vm create` and `az vmss create` now supply auxiliary tokens that were
previously missing.

### Availability-set scheduled-event policy (2.70.0)

`az vm available-set create` and `az vm available-set update` gain
`--additional-scheduled-events`, `--enable-user-reboot-scheduled-events`, and
`--enable-user-redeploy-scheduled-events`.

### Availability-set scheduled-event profiles (2.79.0)

`az vm availability-set update` accepts `--enable-all-instance-down` and
`--scheduled-events-api-version` for the scheduled-events profile.

### Availability-set to VMSS migration (2.78.0)

`az vm availability-set` adds validation, start, cancellation, and conversion
operations for VMSS migration; `az vm migrate-to-vmss` migrates a VM.

### Compute command deprecations (2.71.0)

`--marker` and `--show-next-marker` are deprecated on the shared and community
image-definition and image-version list commands. `az vm list-sizes` is also
deprecated, so new automation should not depend on these interfaces.

### Compute output and option removals (2.69.0)

The gallery-application create/update output field is now
`supportedOSType`, not `supportedOsType`, which is a breaking change for
case-sensitive output consumers. `az vm list-sizes` no longer accepts the
unused `--ids` option.

### Flexible Consumption location discovery (2.71.0)

`az functionapp list-flexconsumption-locations` accepts `--details` to return
more location information and `--runtime` to select a runtime.

### Removed VMSS scheduled-event option (2.75.0)

`az vmss create` and `az vmss update` no longer accept the overlong
`--scheduled-event-additional-publishing-target-event-grid-and-resource-graph`
option. Automation still passing it must be updated.

### Scheduled-events profiles (2.88.0)

`az vm` and `az vmss` create, update, and show operations now surface
scheduled-events profiles through `--scheduled-events-api-version` and
`--enable-all-instance-down`. Availability-set create and show gain the same
support; update already had these options.

### Spot placement command replacement (2.76.0)

Use `az compute-recommender spot-placement-score` in place of
`az compute-recommender spot-placement-recommender`.

### Standard VM security is no longer implicit (2.72.0)

VM and VMSS create/update commands now set `--security-type Standard` only
when the caller explicitly supplies it. Automation that needs Standard in the
request must pass the option rather than relying on CLI injection.

### Standard-policy TVM protection (2.73.0)

`az backup protection enable-for-vm` now supports protecting a TVM with a
standard policy.

### VM and scale-set capabilities (2.69.0)

VM scale-set create/update gains `--zone-balance`; scale now supports edge
zones. Scale-set create and `az vmss encryption enable` gain
`--encryption-identity` for Azure disk-encryption identity, and VM/VMSS
creation automatically installs the guest-attestation extension when the
security type is `ConfidentialVM`.

### VM and VMSS default size (2.87.0)

When no size is supplied, `az vm create` and `az vmss create` now default to
`Standard_D2s_v5` instead of `Standard_DS1_v2`. Pass `--size` or `--vm-sku`
explicitly when provisioning must remain stable across CLI versions.

### VM and VMSS metadata-endpoint controls (2.72.0)

VM and VMSS create/update commands gain `--wire-server-mode` with
`--wire-server-access-control-profile-reference-id`, plus `--imds-mode` with
`--imds-access-control-profile-reference-id`. They also accept
`--key-incarnation-id`.

### VM and VMSS ProxyAgent installation (2.80.0)

`az vm` and `az vmss` create and update accept
`--add-proxy-agent-extension` to control whether the ProxyAgent Extension is
installed implicitly.

### VM scale-set security posture arguments (2.68.0)

`az vmss create` and `az vmss update` gain
`--security-posture-reference-is-overridable`. The existing
`--security-posture-reference-exclude-extensions` option now receives a
string list, so callers can pass multiple excluded extensions.

### VM scheduled-event policy (2.68.0)

`az vm create` and `az vm update` gain `--additional-scheduled-events`,
`--enable-user-reboot-scheduled-events`, and
`--enable-user-redeploy-scheduled-events` for configuring scheduled-event
policy.

### VM zone movement and force deallocation (2.87.0)

VM create/update operations accept `--zone-movement`, and existing VMs can be
moved across zones through `az vm update`. `az vm deallocate
--force-deallocate` performs a forced deallocation.

### VMSS automatic repairs (2.76.0)

`az vmss create --enable-automatic-repairs` configures the scale set's
automatic-repairs policy during creation.

### VMSS automatic zone placement (2.85.0)

`az vmss create --zone-placement-policy Auto` can constrain automatic
placement with `--include-zones`, `--exclude-zones`, and `--max-zone-count`;
VMSS update also gains `--max-zone-count`. Create/update can enforce a
per-zone instance percentage with `--instance-percent-policy` and
`--value-max-instance-percent-per-zone`.

### VMSS resiliency views (2.82.0)

`az vmss list-instances --resiliency-view` includes each instance's resiliency
status, while `az vmss get-resiliency-view` retrieves the per-instance view
directly.

### VMSS zone balancing and instance-mix ranking (2.73.0)

VMSS create/update accepts `--enable-automatic-zone-balancing`,
`--automatic-zone-balancing-strategy`, and
`--automatic-zone-balancing-behavior`. It also accepts `--skuprofile-rank` as
a list of ranks for the instance-mix SKU profile's VM sizes.

### VMSS zone-placement updates (2.88.0)

`az vmss update` now accepts `--zone-placement-policy`, `--include-zones`,
and `--exclude-zones`, extending the automatic zone-placement controls to
existing scale sets.

## Disks, images, and backup

### Backup reconfiguration to another vault (2.78.0)

The new `az backup protection reconfigure` command can reconfigure protection
to use an alternate vault.

### Confidential disks and instant-access snapshots (2.77.0)

`az disk create` and `az disk grant-access` now support Confidential VM OS
disks. `az snapshot create --instant-access-duration-minutes` sets the instant
access duration for Premium SSD v2 and Ultra Disk snapshots.

### Confidential-VM disk restore encryption (2.76.0)

`az backup restore restore-disks --cvm-os-des-id` selects the Disk Encryption
Set used for a confidential VM's restored OS disk.

### Deleted Backup vault recovery (2.79.0)

The new `az backup vault deleted-vault` command group can list and undelete
deleted Backup vaults.

### Disk and snapshot output schemas (2.68.0)

The output fields from `az disk` and `az snapshot` have breaking changes to
align them with the backend service. Automation that parses their JSON or
table output must be checked against the 2.68.0 shape.

### Disk PATCH updates and explicit Standard security (2.71.0)

`az disk config update` can change disk size in GB through a PATCH operation.
VM and VM scale-set create/update commands also allow `Standard` as an
explicit security type.

### Fully cached ephemeral OS disks (2.87.0)

VM and VM scale-set creation accept
`--ephemeral-os-disk-enable-full-caching` to use full caching with an
ephemeral OS disk.

### Implicit disk creation during attach (2.76.0)

`az vm disk attach` can create a disk implicitly from snapshots or disk
restore points via `--source-snapshots-or-disks` and
`--source-disk-restore-point`; the implicit disk's size and SKU can also be
set.

### Instant-access restore points (2.85.0)

Restore-point collection create/update accepts `--instant-access`, and
`az restore-point create --instant-access-duration` sets the instant-access
duration.

### Managed-disk security and availability policies (2.78.0)

`az disk create` and `update` accept `--supported-security-option` and
`--action-on-disk-delay`.

### Names for disks created during attach (2.78.0)

`az vm disk attach` accepts `--new-names-of-source-snapshots-or-disks` and
`--new-names-of-source-disk-restore-point` to name newly created disks.

### New Azure Backup workload support (2.74.0)

The `az backup container`, `item`, `policy`, and `protection` command groups
now support ASE backup operations, and `az backup` supports HANA Snapshot.

### No-zone disk restores (2.70.0)

`az backup restore restore-disks --target-zone` now accepts `NoZone` as a
valid restore target.

### Shared Image Gallery in-VM access controls (2.76.0)

The new `az sig in-vm-access-control-profile` and
`az sig in-vm-access-control-profile-version` groups manage in-VM access
control profiles and their versions.

### Shared Image Gallery managed identity (2.86.0)

`az sig create` can configure a Shared Image Gallery's managed service
identity, `az sig show` returns it, and the new `az sig identity` command
group manages it after creation.

### Shared Image Gallery pagination (2.73.0)

The community/shared image-definition and image-version list commands replace
their old pagination interface with `--max-items` and `--next-token`.

### Shared Image Gallery VHD property remapping (2.72.0)

In a breaking change, `az sig image-version` maps
`--os-vhd-storage-account` to
`properties.storageProfile.osDiskImage.source.storageAccountId` and
`--data-vhds-storage-accounts` to
`properties.storageProfile.dataDiskImages.source.storageAccountId`.

### Shared-image deletion guard (2.71.0)

`az sig image-version create` and `az sig image-version update` accept
`--block-deletion-before-end-of-life` to prevent deletion before the image
version's end-of-life date.

### VM data-disk performance settings (2.84.0)

`az vm create` accepts `--data-disk-mbps` and `--data-disk-iops` to set MBPS
and IOPS for data disks during creation.

### VM disk-encryption identity (2.68.0)

`az vm create` accepts `--encryption-identity` to select the managed identity
used for Azure disk encryption. The same option on `az vm encryption enable`
sets or updates that identity for an existing VM.

### VM zone placement and disk alignment (2.72.0)

`az vm create` gains `--zone-placement-policy`, `--include-zones`, and
`--exclude-zones` for zonal placement. VM create/update also gains
`--align-regional-disks-to-vm-zone` to convert attached regional disks to
zonal disks.

## App Service and Functions

### App Service asynchronous scaling (2.78.0)

`az appservice plan create` and `update` accept `--async-scaling-enabled`.

### App Service command changes (2.69.0)

`az functionapp deployment slot create` gains `--https-only`. On Linux,
`az webapp list-runtimes` no longer returns JBoss `_byol` entries, so scripts
that select those runtime identifiers must be updated.

### App Service deployment diagnostics and conversion (2.86.0)

`az webapp up` and `az webapp deploy` accept `--enriched-errors` to return
detailed deployment-failure logs. `az webapp sitecontainers convert` can now
convert Docker Compose multi-container apps to Sitecontainers mode.

### App Service domain-label scope and container conversion (2.76.0)

`az webapp create` accepts `--domain-name-scope` for DNL scope selection, and
`az webapp sitecontainers convert` switches an app between sitecontainers and
classic configuration.

### App Service hostname scope and release channels (2.85.0)

`az logicapp create` and `az webapp up` accept `--domain-name-scope` to
select the uniqueness scope for the default hostname. `az webapp update`
accepts `--platform-release-channel` to set the app's platform release
channel.

### App Service lifecycle and zoning (2.73.0)

App Service Environment create/update/delete no longer supports ASEv2.
`az functionapp plan update` can now update zone redundancy for Flex plans.

### App Service output and runtime discovery (2.84.0)

`az webapp config access-restriction show` now always returns values in camel
case, so consumers of its output must use that casing. `az webapp list
runtimes` no longer relies on hardcoded runtime lists and includes previously
missing Java versions.

### App Service plan operating-system default (2.88.0)

Without an explicit `--hyper-v`, `az appservice plan create` now defaults to
Linux. Pass `--is-linux false` when creating a Windows App Service plan.

### App Service VNet routing behavior (2.84.0)

For API version `2024-11-01`, Web App create/configuration and Web App or
Function App VNet-integration commands now use the site-level outbound VNet
routing property.

### Consumption-to-Flex function migration (2.77.0)

The new `az functionapp flex-migration` command group supports migrating CV1
function apps to Flex.

### Deployment-slot VNet inheritance (2.71.0)

`az webapp deployment slot create` now gives a new slot the same VNet
integration settings as its source slot, matching Portal-created slots.

### Flex Consumption certificates (2.88.0)

`az functionapp config ssl` now supports site-scoped certificates for Flex
Consumption. `az functionapp flex-migration` can also migrate Linux
Consumption apps that have certificates.

### Function App update strategies (2.87.0)

`az functionapp update-strategy config set` sets or updates a Function App's
update-strategy configuration, while `config show` retrieves it.

### Linux App Service plan default (2.86.0)

When `--sku` is omitted for a Linux web app, `az appservice plan create` now
defaults to `P0V3`. The command also recognizes the `PREMIUM0V3` tier for
elastic scale, so automation that depends on another plan size must pass it
explicitly.

### Linux container startup logs (2.87.0)

The new `az webapp log startup` commands list and display Linux container
startup logs.

### Linux Web App site containers and Kudu warm-up (2.70.0)

Linux web apps gain the `az webapp sitecontainers` command group. Deployment
through `az webapp up`, `az webapp deploy`, or
`az webapp deployment source config-zip` can use `--enable-kudu-warmup` to
warm Kudu before deploying.

### Managed-instance-aware App Service locations (2.82.0)

`az appservice list-locations` accepts `--managed-instance-enabled` when
discovering locations that support managed instances.

### Site-scoped Web App certificates (2.87.0)

`az webapp create --site-scoped-certs` controls whether site-scoped
certificates are enabled for a new app.

### Structured Web App runtime discovery (2.87.0)

In a breaking output change, `az webapp list-runtimes` now returns objects
with `os`, `runtime`, `version`, `config`, `support`, and `end_of_life`
fields instead of a flat string list. Use the new `--runtime` and `--support`
filters; `--linux` and `--show-runtime-details` have been removed.

### Web App transport encryption (2.84.0)

`az webapp create` and `az webapp update` accept
`--end-to-end-encryption-enabled` for encryption between the front end and
workers. Creation also accepts `--min-tls-version` and
`--min-tls-cipher-suite`.

### Web App worker-count validation (2.74.0)

`az webapp config set` no longer performs CLI validation of the number of
workers, so that check no longer rejects the request before it reaches Azure.

### Zone-redundant Elastic Premium Functions (2.80.0)

`az functionapp plan create` now supports zone redundancy for Elastic Premium
SKUs.

## Batch, AI, ARO, HDInsight, and Service Fabric

### AI Foundry command groups (2.80.0)

The `az cognitiveservices account connection`,
`az cognitiveservices account project`, and
`az cognitiveservices account project connection` groups manage AI Foundry
resources, and `az cognitiveservice agent` is a new command group.

### ARO VM SKU selection (2.71.0)

`az aro create` uses an updated VM SKU selection aligned with current best
practices; automation that depends on a particular SKU should choose it
explicitly.

### Batch pool argument removals (2.80.0)

`az batch pool create` no longer accepts `--target-communication` or
`--resource-tags`; pool `reset` and `set` also drop
`--target-communication`.

### Cognitive Services project management and kind changes (2.78.0)

`az cognitiveservices account create` accepts `--allow-project-management`,
and `account update --kind` supports OpenAI-to-AIServices kind changes and back.

### Expanded Batch task and JSON configuration (2.69.0)

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

### HDInsight credential operations (2.79.0)

`az hdinsight credentials show` retrieves current cluster credentials, and
`az hdinsight credentials update` changes them.

### HDInsight Entra and managed-identity storage (2.79.0)

`az hdinsight create` can create Entra-enabled clusters and clusters using
WASB with a managed identity.

### Hosted AI Foundry agents (2.82.0)

`az cognitiveservices agent create` can create and deploy a hosted agent in
AI Foundry.

### Hosted-agent log streaming (2.83.0)

`az cognitiveservices agent logs show` streams console logs for hosted agents.
Agent `create` and `start` accept `--show-logs`, and `start` also accepts
`--timeout`.

### Removed Batch commands and options (2.69.0)

The deprecated `az batch certificate create/list/show/delete`,
`az batch node reimage`, and `az batch node remote-desktop` commands are
removed. Batch pool creation also removes `--application-licenses`,
`--certificate-references`, `--os-family`, and `--os-version`; pool set/reset
removes `--certificate-references`.

### Service Fabric cluster names from parameter files (2.77.0)

When a parameters file supplies `cluster_name`, `az sf cluster create` now
uses that value.

### Service Fabric managed-cluster controls (2.76.0)

Managed-cluster network security rules accept `--source-addr-prefix`,
`--dest-addr-prefix`, `--source-port-range`, and `--dest-port-range`.
`az sf managed-node-type update` can also change `--vm-size` and `--tags`.

### Service Fabric update argument removals (2.80.0)

`az sf managed-application update` drops `--service-type-policy`,
`--upgrade-replica-set-check-timeout`, `--max-porcent-unhealthy-partitions`,
`--max-porcent-unhealthy-replicas`, `--max-porcent-unhealthy-services`, and
`--max-porcent-unhealthy-apps`. `az sf application update` drops
`--service-type-policy`, `--upgrade-replica-set-check-timeout`,
`--instance-close-duration`, `--consider-warning-as-error`,
`--max-percent-unhealthy-partitions`, `--max-percent-unhealthy-replicas`, and
`--max-percent-unhealthy-deployed-applications`.
