# Data Storage And Backup

Use this reference for PostgreSQL, MySQL, SQL, Cosmos DB, Storage, Azure Files, NetApp Files, Redis, and Backup.

## Announced PostgreSQL flexible-server interface changes

_Azure CLI `2.86.0`._

Create, replica-create, restore, geo-restore, and revive-dropped operations
now warn of an upcoming breaking behavioral change involving network
resources. Flexible-server creation also announces the deprecation of
`--cluster-option`; treat both interfaces as transitional in automation.

## Azure Files NFS operations

_Azure CLI `2.71.0`._

The storage share, directory, and file commands support NFS file shares, and
`az storage file hard-link create` creates hard links for NFS files.

## Azure Files restore after source-account deletion

_Azure CLI `2.77.0`._

`az backup restore restore-azurefileshare` now supports restores whose source
storage account has been deleted by including the required source resource ID
in the request.

## Azure Files Vault Standard backup policies

_Azure CLI `2.69.0`._

The `az backup` command group now supports AFS Vault Standard policies.

## Azure NetApp Files clone splitting

_Azure CLI `2.78.0`._

`az netappfiles volume splitclonefromparent` splits a clone from its parent;
volume creation also accepts `--grow-pool-clone-split`.

## Azure NetApp Files configuration changes

_Azure CLI `2.73.0`._

`az volume-group create` no longer requires `--proximity-placement-group`.
NetApp account create/update accepts `--federated-client-id` for cross-tenant
customer-managed keys and `--nfs-v4-id-domain` for NFSv4 user-ID mapping.

## Azure NetApp Files cool-access tiering

_Azure CLI `2.70.0`._

`az netappfiles volume create` and `az netappfiles volume update` accept
`--cool-access-tiering-policy`.

## Azure NetApp Files encryption-key transitions

_Azure CLI `2.70.0`._

`az netappfiles account change-key-vault` changes the Key Vault or Managed HSM
used to encrypt an account's volumes. `get-key-vault-status` supplies Key Vault
information for `transitiontocmk`, which transitions all volumes in a VNet to
a different Microsoft-managed or Key Vault key source; it fails when targeted
volumes share an encryption sibling set with another account's volumes.

## Azure NetApp Files ransomware and quota reports

_Azure CLI `2.85.0`._

NetApp Files volume create/update accepts
`--desired-ransomware-protection-state`. The new
`az netappfiles volume ransomware-report` group exposes advanced ransomware
reports, while `az netappfiles volume list-quota-report` lists volume quota
reports.

## Backup reconfiguration to another vault

_Azure CLI `2.78.0`._

The new `az backup protection reconfigure` command can reconfigure protection
to use an alternate vault.

## Capability-gated PostgreSQL 17 upgrades

_Azure CLI `2.72.0`._

`az postgres flexible-server upgrade --version` now checks the server
capability API and permits PostgreSQL 17 when that capability is available.

## Cosmos DB burst capacity

_Azure CLI `2.68.0`._

`az cosmosdb create` and `az cosmosdb update` accept
`--enable-prpp-autoscale` to enable or disable the burst-capacity feature.

## Cosmos DB fleets and local authentication

_Azure CLI `2.82.0`._

`az cosmosdb fleet` is the new command group for Cosmos DB fleets. Account
create and update also accept `--disable-local-auth` so local authentication
can be disabled.

## Cosmos DB region offlining

_Azure CLI `2.70.0`._

`az cosmosdb offline-region` can take a region in a Cosmos DB account offline.

## Cosmos DB restore validation behavior

_Azure CLI `2.76.0`._

`az cosmosdb restore` no longer performs the CLI-side validations that could
time out for large restores or report incorrect errors; restore requests now
proceed without those checks.

## Cosmos DB SQL full-text policies

_Azure CLI `2.74.0`._

`az cosmosdb sql container` now supports Full Text Policy configuration.

## Cross-subscription MySQL operations

_Azure CLI `2.83.0`._

MySQL flexible-server `restore`, `geo-restore`, and `replica create` now
support targeting a different subscription.

## Cross-subscription SQL geo-replication

_Azure CLI `2.75.0`._

`az sql db replica create` can specify the partner subscription ID when
creating a cross-subscription geo-replica.

## Deleted Backup vault recovery

_Azure CLI `2.79.0`._

The new `az backup vault deleted-vault` command group can list and undelete
deleted Backup vaults.

## Deleting on-demand MySQL backups

_Azure CLI `2.82.0`._

`az mysql flexible-server backup delete` can delete an on-demand backup.

## DMS location and NetApp endpoint type

_Azure CLI `2.80.0`._

`az dms project create` no longer requires `--location`. NetApp volume create
and update no longer accept the read-only `--endpoint-type` argument.

## Fabric workspaces in Cosmos DB ACL bypasses

_Azure CLI `2.84.0`._

`az cosmosdb update --network-acl-bypass-resource-ids` now accepts Microsoft
Fabric workspace resource IDs.

## File-service transport encryption requirements

_Azure CLI `2.83.0`._

`az storage account file-service-properties update` adds
`--require-smb-encryption-in-transit` and
`--require-nfs-encryption-in-transit`.

## Flexible Azure NetApp Files throughput

_Azure CLI `2.78.0`._

Pool and volume creation accept `Flexible` as a service level, and
`az netappfiles pool create` accepts `--custom-throughput-mibps`.

## Geo-priority replication

_Azure CLI `2.79.0`._

Storage-account create and update accept
`--enable-blob-geo-priority-replication` for Geo SLA. Object-replication
policy create and update accept `--priority-replication` for priority
replication.

## HDInsight Entra and managed-identity storage

_Azure CLI `2.79.0`._

`az hdinsight create` can create Entra-enabled clusters and clusters using
WASB with a managed identity.

## Managed-identity OAuth for SMB shares

_Azure CLI `2.78.0`._

`az storage account create` and `update` accept `--enable-smb-oauth`, allowing
managed identities to access SMB shares through OAuth.

## MySQL 8.4 upgrades

_Azure CLI `2.77.0`._

`az mysql flexible-server upgrade --version 8.4` is now supported.

## MySQL Accelerated Logs

_Azure CLI `2.78.0`._

`az mysql flexible-server create` and `update` support Accelerated Logs for
the GeneralPurpose tier.

## MySQL backup and accelerated restore controls

_Azure CLI `2.72.0`._

MySQL flexible-server creation accepts `--backup-interval`. Restore accepts
`--faster-restore` to enable automatic IOPS scaling, and replica creation
accepts `--faster-provisioning` for the same behavior while provisioning.

## MySQL backup interval updates

_Azure CLI `2.76.0`._

MySQL flexible-server create/update exposes the revised
`--storage-redundancy` option and `--backup-interval`; unlike the earlier
create-only support, update can now set the backup interval.

## MySQL BC storage-redundancy default

_Azure CLI `2.74.0`._

`az mysql flexible-server create` now defaults storage redundancy to local
redundancy for BC SKUs; pass the redundancy explicitly when provisioning must
not inherit this changed default.

## MySQL flexible-server default changes

_Azure CLI `2.71.0`._

In a breaking change, `az mysql flexible-server create` changes the defaults
for both `--auto-scale-iops` and `--version`. Reproducible provisioning should
pass both values explicitly rather than inheriting the CLI defaults.

```bash
az mysql flexible-server create --resource-group "$RESOURCE_GROUP" \
  --name "$SERVER" --auto-scale-iops "$AUTO_SCALE_IOPS" \
  --version "$MYSQL_VERSION"
```

## MySQL flexible-server storage redundancy

_Azure CLI `2.68.0`._

`--storage-redundancy` is available on flexible-server create, restore,
replica create, and geo restore to request HA storage with zone redundancy.

## MySQL storage-redundancy argument removed

_Azure CLI `2.87.0`._

MySQL flexible-server backup create, restore, geo-restore, and replica
operations no longer accept `--storage-redundancy`; remove it from scripts
before upgrading the CLI.

## NetApp Files Cache and Bucket resources

_Azure CLI `2.87.0`._

The new `az netappfiles cache` and `az netappfiles volume bucket` command
groups manage Cache and Bucket resources.

## NetApp Files subvolume deprecations

_Azure CLI `2.88.0`._

The `az netappfiles subvolume` command group is deprecated, as is
`--enable-subvolumes` on NetApp Files volume create and update. Both are
announced for removal in a future release.

## NetApp Files volume interface changes

_Azure CLI `2.87.0`._

For `az netappfiles volume create`, the default for `--network-features` is
now `Standard`. The `az netappfiles volume update
--remote-volume-resource-id` option is deprecated.

## NetApp Files volume-group networking and replication filtering

_Azure CLI `2.81.0`._

`az netappfiles volume-group create` accepts `--network-features` for volume
groups. `az netappfiles volume replication list` accepts `--exclude` to omit
deleted replications.

## New Azure Backup workload support

_Azure CLI `2.74.0`._

The `az backup container`, `item`, `policy`, and `protection` command groups
now support ASE backup operations, and `az backup` supports HANA Snapshot.

## NFS file listing

_Azure CLI `2.79.0`._

`az storage file list` now handles NFS shares; `--include` remains unsupported
for those shares.

## NFS file-share symbolic links

_Azure CLI `2.78.0`._

`az storage file symoblic-link create` and `show` manage symbolic links on
NFS file shares.

## OAuth for Azure Files batch transfers

_Azure CLI `2.75.0`._

`az storage file upload-batch` and `az storage file download-batch` now
support OAuth login.

## Object-replication metrics

_Azure CLI `2.78.0`._

`az storage account or-policy create` and `update` accept `--enable-metrics`
to enable object-replication metrics.

## Oracle Azure NetApp Files volume groups

_Azure CLI `2.74.0`._

`az netappfiles volume-group create` now supports Oracle in ANF Volume Groups.

## PostgreSQL 11 and 12 end-of-life handling

_Azure CLI `2.75.0`._

`az postgres flexible-server create` extends its end-of-life handling to
PostgreSQL 11 and 12, so provisioning should not rely on creating either
version.

## PostgreSQL authentication during creation

_Azure CLI `2.71.0`._

`az postgres flexible-server create` can add an administrator while
`--active-directory-auth` is enabled. When `--password-auth` is disabled, the
command no longer generates an otherwise unusable password.

## PostgreSQL autonomous tuning and version 18 upgrades

_Azure CLI `2.82.0`._

The `az postgres flexible-server index-tuning` group is deprecated and
redirects to `az postgres flexible-server autonomous-tuning`. Use its
`list-index-recommendations` and `list-table-recommendations` commands;
`az postgres flexible-server upgrade` also supports PostgreSQL 18.

## PostgreSQL cluster, list, and replica arguments

_Azure CLI `2.82.0`._

Elastic-cluster creation has a database-name field that defaults to `None`.
The `backup`, `db`, `firewall-rule`, `identity`, `long-term-retention`,
`microsoft-entra-admin`, `migration`, `parameter`, and `replica` list commands
accept `--ids`; `replica create --name` can choose the read-replica name.

## PostgreSQL command and creation removals

_Azure CLI `2.80.0`._

The Single Server groups `az postgres server`, `az postgres db`, and
`az postgres server-logs` are removed. Flexible-server creation no longer has
a default for `--version` and drops `--create-default-database` and
`--database-name`.

## PostgreSQL command and version removals

_Azure CLI `2.73.0`._

`az postgres flexible-server stop-replication` is removed; use
`az postgres flexible-server replica promote`. Flexible-server create and
upgrade also no longer support PostgreSQL 12.

## PostgreSQL elastic-cluster replicas

_Azure CLI `2.76.0`._

`az postgres flexible-server replica create` and `promote` now support elastic
clusters.

## PostgreSQL flexible-server argument changes

_Azure CLI `2.87.0`._

Flexible-server create/update removes `--high-availability`; use
`--zonal-resiliency` instead. Upgrade no longer constrains `--version` with a
CLI enum, and backup creation no longer requires a backup name because one is
generated automatically.

The `backup`, `db`, `firewall-rule`, `migration`, and `replica` create
commands now consistently use `--name` and `--server-name`; update scripts
whose old parameter names differ.

## PostgreSQL flexible-server creation defaults

_Azure CLI `2.73.0`._

Creation now defaults `--create-default-database` to Disabled and the
PostgreSQL version to 17. The default SKU is selected from the location
capability API, so scripts needing stable choices should pass these values
explicitly.

## PostgreSQL flexible-server elastic clusters

_Azure CLI `2.68.0`._

Create an elastic cluster with `--cluster-option ElasticCluster`, include
elastic clusters in list results with `--show-cluster`, and scale one with
the update command's `--node-count`. The flexible-server `identity` and
`fabric-mirroring` command groups also support system-assigned managed
identity and database mirroring to Fabric.

## PostgreSQL HA storage and mirroring

_Azure CLI `2.82.0`._

For PostgreSQL 17 or later, an HA-enabled flexible server may now start Fabric
mirroring, reversing the earlier HA restriction. Flexible-server create and
update also expose zonal resiliency for HA and allow HA with `PremiumV2_LRS`
storage.

## PostgreSQL index-tuning options

_Azure CLI `2.70.0`._

`az postgres flexible-server index-tuning` gains operations for tuning
options.

## PostgreSQL long-term-retention removal

_Azure CLI `2.85.0`._

The `az postgres flexible-server long-term-retention` command group now
announces its upcoming removal; avoid introducing new automation that
depends on it.

## PostgreSQL multi-tenant identity and maintenance events

_Azure CLI `2.88.0`._

Flexible-server create, restore, geo-restore, and replica create accept
`--federated-client-id` and `--backup-federated-client-id` for multi-tenant
application registration. The new `az postgresql flexible-server
maintenance-event` list, show, apply-now, and reschedule commands manage
maintenance events.

## PostgreSQL network-mode migration

_Azure CLI `2.84.0`._

The new `az postgres flexible-server migrate-network` command migrates a
flexible server's network mode.

## PostgreSQL Premium SSDv2 behavior

_Azure CLI `2.86.0`._

Read-replica creation accepts `--storage-type PremiumV2_LRS`. Increasing the
storage size of a Premium SSDv2 server no longer requires a restart, while
create and upgrade now reject SSDv2 for PostgreSQL versions earlier than 14.

## PostgreSQL restore time and HA mirroring restriction

_Azure CLI `2.69.0`._

`az postgres flexible-server geo-restore` gains `--restore-time`. Fabric
mirroring start/stop/update-databases operations are disabled for HA servers.

## PostgreSQL SSDV2 replica and geo-restore support

_Azure CLI `2.83.0`._

PostgreSQL flexible-server create, georestore, and replica operations now
allow SSDV2 servers to create replicas and perform geo-restores.

## PostgreSQL update prompts and public access

_Azure CLI `2.73.0`._

Some flexible-server update operations now ask for user confirmation, which
changes unattended command behavior. Creation now disables public network
access when its public-access argument is `None`.

## Provisioned Azure Files controls

_Azure CLI `2.70.0`._

`az storage account create --sku` adds `StandardV2_LRS`, `StandardV2_ZRS`,
`PremiumV2_LRS`, and `PremiumV2_ZRS` for provisioned v2 accounts, and
`az storage account file-service-usage` reports file-service usage. Share
create/update gains `--paid-bursting-enabled`,
`--paid-bursting-max-bandwidth-mibps`, and `--paid-bursting-max-iops` for
provisioned v1, plus `--provisioned-bandwidth-mibps` and `--provisioned-iops`
for provisioned v2.

## Redis zoning and system-identity connections

_Azure CLI `2.69.0`._

`az redis create` and `az redis update` gain `--zonal-allocation-policy` for
choosing cache zones. `az webapp connection create redis` gains
`--system-identity`.

## Soft-deleted SQL server lifecycle

_Azure CLI `2.84.0`._

`az sql server create` and `update` accept `--soft-delete-retention-days`.
The new `az sql server deleted-server show` and `list` commands inspect
deleted servers, and `az sql server restore` restores one.

## SQL long-term retention and failover groups

_Azure CLI `2.76.0`._

`az sql ltr-policy set` removes the unused `--access-tier` option, so callers
must stop passing it. `az sql failover-group create` now supports multiple
partner failover groups.

## SQL long-term-retention immutability

_Azure CLI `2.78.0`._

The `az sql db ltr-backup` group adds commands for LTR immutability.

## SQL Managed Instance memory sizing

_Azure CLI `2.82.0`._

`az sql mi create` and `update` can set the managed instance's memory size in
GB.

## SQL serverless-to-provisioned updates

_Azure CLI `2.79.0`._

When `az sql db update` moves a database from serverless to provisioned, it
no longer overwrites the selected service-level objective.

## Storage account and blob-service controls

_Azure CLI `2.87.0`._

Storage account create/update accepts the `Smart` value for `--access-tier`
and adds `--allowed-copy-scope`. Blob-service-properties update can configure
static website enablement, index documents, and the 404 document through
`--enable-static-website`, `--index-document`,
`--default-index-document-path`, and `--error-document-404-path`; object
replication policy create/update also accepts `--tags-replication`.

## Storage failover and listing behavior

_Azure CLI `2.80.0`._

`az storage account failover --failover-type` now accepts `Unplanned`.
`az storage file list` now returns its additional information even when no
protocol is explicitly selected.

## Storage IPv6 endpoints and network rules

_Azure CLI `2.83.0`._

Storage account create/update adds `--publish-ipv6-endpoint`, while
`az storage account network-rule add` and `remove` add `--ipv6-address`.

## Storage SAS expiration actions

_Azure CLI `2.75.0`._

`az storage account create` and `az storage account update` accept
`--sas-expiration-action` as part of the account's SAS policy.

## Storage-account migration confirmation

_Azure CLI `2.73.0`._

`az storage account migration start` now asks for confirmation before
migrating an account between redundancy options.

## Storage-account network security perimeters

_Azure CLI `2.79.0`._

The `az storage account network-security-perimeter-configuration` group adds
`list`, `show`, and `reconcile` operations for network security perimeters.

## Storage-account zone placement

_Azure CLI `2.78.0`._

`az storage account create` and `update` accept `--zones` and
`--zone-placement-policy` for zones and availability-zone pinning.

## User-delegation SAS expansion

_Azure CLI `2.82.0`._

`az storage blob generate-sas`, `az storage container generate-sas`,
`az storage fs generate-sas`, `az storage fs file generate-sas`, and
`az storage fs directory generate-sas` accept `--user-delegation-oid`; the
filesystem-file command is new. `az storage share generate-sas`,
`az storage file generate-sas`, and `az storage queue generate-sas` add that
option together with `--as-user`.

## Versionless SQL TDE keys

_Azure CLI `2.84.0`._

Azure SQL server and database commands now support versionless Transparent
Data Encryption keys.
