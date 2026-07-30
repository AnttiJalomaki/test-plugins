# Storage and Volumes

Use this reference for dynamic host volumes, stateful workloads, CSI event
visibility, scheduler disk capacity, and quota migrations.

## Dynamic host volumes

Nomad can create host volumes through the CLI or API without restarting
clients. Stateful deployments consume them through `volume` and
`volume_mount` blocks:

```shell
nomad volume create ./internal-plugin.volume.hcl
```

The scheduler tracks declared availability, but Nomad does not interpret the
underlying storage. A dynamic host volume may therefore be backed by local
storage or highly available network storage. Match rescheduling and recovery
expectations to the actual backend (source batch `1.10.0`).

Nomad Enterprise can apply Sentinel policy during volume creation, enforce
per-namespace host-volume capacity quotas, and verify the requested node pool
against namespace node-pool configuration.

## Volume CLI and events

`nomad volume status` displays volume capabilities. `nomad volume delete`
accepts a volume ID prefix and a wildcard namespace. Be careful to resolve the
intended namespace and prefix before deleting.

CSI volume and plugin events are present in the event stream. Event consumers
can observe lifecycle and plugin activity without relying exclusively on
polling status endpoints.

## Scheduling capacity

Storage available for scheduling is calculated as:

```text
totalBytes - client.reserved.disk
```

It is no longer based on current free disk space, and the
`unique.storage.bytesfree` attribute is removed (source batch `1.11-upgrade`).
Reserve at least the disk space used by the host operating system. If a
placement decision looks inconsistent with `df`, inspect configured total and
reserved capacity rather than trying to restore the removed attribute.

## Quota schema migration

The quota field `variables_limit` and Go API field
`QuotaSpec.VariablesLimit` are deprecated for removal in 1.12. Use:

```text
region_limit.storage.variables
QuotaSpec.RegionLimit.Storage.Variables
```

The Go API type of `QuotaSpec.RegionLimit` changes from `Resources` to
`QuotaResources`. Update configuration generators, JSON/HCL serializers,
client code, and tests together so schema shape and Go types remain aligned.
