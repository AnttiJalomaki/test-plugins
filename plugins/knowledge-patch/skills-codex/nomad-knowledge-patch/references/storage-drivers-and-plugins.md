# Storage, Drivers, and Plugins

## Dynamic host volumes

Nomad can create host volumes through the CLI or API without restarting
clients, and stateful deployments can use them:

```shell
nomad volume create ./internal-plugin.volume.hcl
```

Jobs consume the volume with `volume` and `volume_mount` blocks. The scheduler
tracks availability but does not interpret the underlying storage; a volume
can be backed by node-local storage or highly available network storage.

Enterprise volume governance can use Sentinel at creation time, impose
per-namespace host-volume capacity quotas, and validate a requested node pool
against the namespace's configured node pools.

Dynamic host volumes and governance are from batch `1.10.0`.

## Volume CLI, API, and events

- CSI volume and CSI plugin events appear in the event stream.
- `nomad volume status` shows volume capabilities.
- `nomad volume delete` accepts a volume ID prefix and a wildcard namespace.

These visibility and CLI changes are from batch `1.10.0`.

## External plugin loading

An executable in `plugin_dir` runs only when a matching `plugin` configuration
block exists. Nomad skips unconfigured plugins. This explicit-loading behavior
is from batch `1.10.0`.

## Removed remote task-driver interface

Task drivers no longer support remote tasks. Custom drivers using that
interface must be redesigned before moving to the behavior in batch `1.10.0`.

## Docker and raw_exec configuration

The Docker driver plugin accepts `image_pull_timeout`.

The `raw_exec` driver accepts `denied_envvars` in both driver and task
configuration. On Windows, it can also select the task user.

These additions are from batch `1.10.0`.

## QEMU machine configuration

Nomad 1.11.1 adds:

```hcl
config {
  emulator     = "qemu-system-x86_64"
  machine_type = "pc"
}
```

Those values are the defaults. The `kvm` accelerator no longer forces the
machine type to `host`. A `resources.cores` value supplies `-smp` only when the
user has not passed a custom `-smp` flag.

In Nomad 1.11.2, filesystem environment variables exposed by the QEMU driver
contain host file paths, replacing relative container paths such as `/alloc`
and `/local`. Update jobspecs that use these variables.

Both QEMU changes are from batch `1.11-upgrade`.

## Executor failure status

Executor failures in the `exec`, `raw_exec`, `java`, and `qemu` task drivers
report exit code `-1` (batch `1.10.0`).

## Secrets plugin timeout

Secrets plugin execution times out after 60 seconds. Use that limit when
implementing and diagnosing secret-provider plugins. This behavior is from
batch `2.0.0`.
