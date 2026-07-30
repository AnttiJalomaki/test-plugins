# Compute And Images

Use this reference for VMs, VM scale sets, disks, snapshots, scheduled events, galleries, and restore points.

## ARO VM SKU selection

_Azure CLI `2.71.0`._

`az aro create` uses an updated VM SKU selection aligned with current best
practices; automation that depends on a particular SKU should choose it
explicitly.

## Automatic VM application upgrades

_Azure CLI `2.81.0`._

`az vm application set` and `az vmss application set` accept
`--enable-automatic-upgrade` to enable automatic application upgrades.

## Auxiliary tokens during VM creation

_Azure CLI `2.73.0`._

`az vm create` and `az vmss create` now supply auxiliary tokens that were
previously missing.

## Availability-set scheduled-event policy

_Azure CLI `2.70.0`._

`az vm available-set create` and `az vm available-set update` gain
`--additional-scheduled-events`, `--enable-user-reboot-scheduled-events`, and
`--enable-user-redeploy-scheduled-events`.

## Availability-set scheduled-event profiles

_Azure CLI `2.79.0`._

`az vm availability-set update` accepts `--enable-all-instance-down` and
`--scheduled-events-api-version` for the scheduled-events profile.

## Availability-set to VMSS migration

_Azure CLI `2.78.0`._

`az vm availability-set` adds validation, start, cancellation, and conversion
operations for VMSS migration; `az vm migrate-to-vmss` migrates a VM.

## Compute command deprecations

_Azure CLI `2.71.0`._

`--marker` and `--show-next-marker` are deprecated on the shared and community
image-definition and image-version list commands. `az vm list-sizes` is also
deprecated, so new automation should not depend on these interfaces.

## Compute output and option removals

_Azure CLI `2.69.0`._

The gallery-application create/update output field is now
`supportedOSType`, not `supportedOsType`, which is a breaking change for
case-sensitive output consumers. `az vm list-sizes` no longer accepts the
unused `--ids` option.

## Confidential disks and instant-access snapshots

_Azure CLI `2.77.0`._

`az disk create` and `az disk grant-access` now support Confidential VM OS
disks. `az snapshot create --instant-access-duration-minutes` sets the instant
access duration for Premium SSD v2 and Ultra Disk snapshots.

## Confidential-VM disk restore encryption

_Azure CLI `2.76.0`._

`az backup restore restore-disks --cvm-os-des-id` selects the Disk Encryption
Set used for a confidential VM's restored OS disk.

## Disk and snapshot output schemas

_Azure CLI `2.68.0`._

The output fields from `az disk` and `az snapshot` have breaking changes to
align them with the backend service. Automation that parses their JSON or
table output must be checked against the 2.68.0 shape.

## Disk PATCH updates and explicit Standard security

_Azure CLI `2.71.0`._

`az disk config update` can change disk size in GB through a PATCH operation.
VM and VM scale-set create/update commands also allow `Standard` as an
explicit security type.

## Fully cached ephemeral OS disks

_Azure CLI `2.87.0`._

VM and VM scale-set creation accept
`--ephemeral-os-disk-enable-full-caching` to use full caching with an
ephemeral OS disk.

## Implicit disk creation during attach

_Azure CLI `2.76.0`._

`az vm disk attach` can create a disk implicitly from snapshots or disk
restore points via `--source-snapshots-or-disks` and
`--source-disk-restore-point`; the implicit disk's size and SKU can also be
set.

## Instant-access restore points

_Azure CLI `2.85.0`._

Restore-point collection create/update accepts `--instant-access`, and
`az restore-point create --instant-access-duration` sets the instant-access
duration.

## Managed-disk security and availability policies

_Azure CLI `2.78.0`._

`az disk create` and `update` accept `--supported-security-option` and
`--action-on-disk-delay`.

## Names for disks created during attach

_Azure CLI `2.78.0`._

`az vm disk attach` accepts `--new-names-of-source-snapshots-or-disks` and
`--new-names-of-source-disk-restore-point` to name newly created disks.

## No-zone disk restores

_Azure CLI `2.70.0`._

`az backup restore restore-disks --target-zone` now accepts `NoZone` as a
valid restore target.

## PostgreSQL disk-tier restriction and legacy HA updates

_Azure CLI `2.77.0`._

Premium SSD v2 can no longer be used with the Burstable compute tier by
`az postgres flexible-server create`, `update`, or `restore`. For existing
PostgreSQL 11 and 12 servers, `az postgres flexible-server update` now
bypasses Fabric mirroring validation so that high-availability status can be
changed.

## Removed VMSS scheduled-event option

_Azure CLI `2.75.0`._

`az vmss create` and `az vmss update` no longer accept the overlong
`--scheduled-event-additional-publishing-target-event-grid-and-resource-graph`
option. Automation still passing it must be updated.

## Scheduled-events profiles

_Azure CLI `2.88.0`._

`az vm` and `az vmss` create, update, and show operations now surface
scheduled-events profiles through `--scheduled-events-api-version` and
`--enable-all-instance-down`. Availability-set create and show gain the same
support; update already had these options.

## Shared Image Gallery in-VM access controls

_Azure CLI `2.76.0`._

The new `az sig in-vm-access-control-profile` and
`az sig in-vm-access-control-profile-version` groups manage in-VM access
control profiles and their versions.

## Shared Image Gallery managed identity

_Azure CLI `2.86.0`._

`az sig create` can configure a Shared Image Gallery's managed service
identity, `az sig show` returns it, and the new `az sig identity` command
group manages it after creation.

## Shared Image Gallery pagination

_Azure CLI `2.73.0`._

The community/shared image-definition and image-version list commands replace
their old pagination interface with `--max-items` and `--next-token`.

## Shared Image Gallery VHD property remapping

_Azure CLI `2.72.0`._

In a breaking change, `az sig image-version` maps
`--os-vhd-storage-account` to
`properties.storageProfile.osDiskImage.source.storageAccountId` and
`--data-vhds-storage-accounts` to
`properties.storageProfile.dataDiskImages.source.storageAccountId`.

## Snapshot virtual-directory access

_Azure CLI `2.71.0`._

`az storage share create` accepts
`--enable-snapshot-virtual-directory-access` for snapshot virtual-directory
access.

## Standard VM security is no longer implicit

_Azure CLI `2.72.0`._

VM and VMSS create/update commands now set `--security-type Standard` only
when the caller explicitly supplies it. Automation that needs Standard in the
request must pass the option rather than relying on CLI injection.

## Standard-policy TVM protection

_Azure CLI `2.73.0`._

`az backup protection enable-for-vm` now supports protecting a TVM with a
standard policy.

## VM and scale-set capabilities

_Azure CLI `2.69.0`._

VM scale-set create/update gains `--zone-balance`; scale now supports edge
zones. Scale-set create and `az vmss encryption enable` gain
`--encryption-identity` for Azure disk-encryption identity, and VM/VMSS
creation automatically installs the guest-attestation extension when the
security type is `ConfidentialVM`.

## VM and VMSS default size

_Azure CLI `2.87.0`._

When no size is supplied, `az vm create` and `az vmss create` now default to
`Standard_D2s_v5` instead of `Standard_DS1_v2`. Pass `--size` or `--vm-sku`
explicitly when provisioning must remain stable across CLI versions.

## VM and VMSS metadata-endpoint controls

_Azure CLI `2.72.0`._

VM and VMSS create/update commands gain `--wire-server-mode` with
`--wire-server-access-control-profile-reference-id`, plus `--imds-mode` with
`--imds-access-control-profile-reference-id`. They also accept
`--key-incarnation-id`.

## VM and VMSS ProxyAgent installation

_Azure CLI `2.80.0`._

`az vm` and `az vmss` create and update accept
`--add-proxy-agent-extension` to control whether the ProxyAgent Extension is
installed implicitly.

## VM data-disk performance settings

_Azure CLI `2.84.0`._

`az vm create` accepts `--data-disk-mbps` and `--data-disk-iops` to set MBPS
and IOPS for data disks during creation.

## VM disk-encryption identity

_Azure CLI `2.68.0`._

`az vm create` accepts `--encryption-identity` to select the managed identity
used for Azure disk encryption. The same option on `az vm encryption enable`
sets or updates that identity for an existing VM.

## VM scale-set security posture arguments

_Azure CLI `2.68.0`._

`az vmss create` and `az vmss update` gain
`--security-posture-reference-is-overridable`. The existing
`--security-posture-reference-exclude-extensions` option now receives a
string list, so callers can pass multiple excluded extensions.

## VM scheduled-event policy

_Azure CLI `2.68.0`._

`az vm create` and `az vm update` gain `--additional-scheduled-events`,
`--enable-user-reboot-scheduled-events`, and
`--enable-user-redeploy-scheduled-events` for configuring scheduled-event
policy.

## VM zone movement and force deallocation

_Azure CLI `2.87.0`._

VM create/update operations accept `--zone-movement`, and existing VMs can be
moved across zones through `az vm update`. `az vm deallocate
--force-deallocate` performs a forced deallocation.

## VM zone placement and disk alignment

_Azure CLI `2.72.0`._

`az vm create` gains `--zone-placement-policy`, `--include-zones`, and
`--exclude-zones` for zonal placement. VM create/update also gains
`--align-regional-disks-to-vm-zone` to convert attached regional disks to
zonal disks.

## VMSS automatic repairs

_Azure CLI `2.76.0`._

`az vmss create --enable-automatic-repairs` configures the scale set's
automatic-repairs policy during creation.

## VMSS automatic zone placement

_Azure CLI `2.85.0`._

`az vmss create --zone-placement-policy Auto` can constrain automatic
placement with `--include-zones`, `--exclude-zones`, and `--max-zone-count`;
VMSS update also gains `--max-zone-count`. Create/update can enforce a
per-zone instance percentage with `--instance-percent-policy` and
`--value-max-instance-percent-per-zone`.

## VMSS resiliency views

_Azure CLI `2.82.0`._

`az vmss list-instances --resiliency-view` includes each instance's resiliency
status, while `az vmss get-resiliency-view` retrieves the per-instance view
directly.

## VMSS zone balancing and instance-mix ranking

_Azure CLI `2.73.0`._

VMSS create/update accepts `--enable-automatic-zone-balancing`,
`--automatic-zone-balancing-strategy`, and
`--automatic-zone-balancing-behavior`. It also accepts `--skuprofile-rank` as
a list of ranks for the instance-mix SKU profile's VM sizes.

## VMSS zone-placement updates

_Azure CLI `2.88.0`._

`az vmss update` now accepts `--zone-placement-policy`, `--include-zones`,
and `--exclude-zones`, extending the automatic zone-placement controls to
existing scale sets.
