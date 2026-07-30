# Drivers, Plugins, and Platforms

Use this reference for external plugin activation, custom task drivers,
Docker, raw_exec, QEMU, executor failures, secrets plugins, and platform
support.

## External plugin activation

An executable in `plugin_dir` runs only when a matching `plugin` configuration
block exists. Unconfigured executables are skipped (source batch `1.10.0`).
Inventory the directory and configuration together; copying a binary alone no
longer activates it.

## Removed remote-task interface

Task drivers no longer support remote tasks. This is a breaking change for
custom drivers that implemented that interface. Remove the remote-task path
and redesign execution around supported local task-driver behavior before
upgrading.

## Docker and raw_exec configuration

The Docker driver plugin accepts `image_pull_timeout`; use it to set the image
pull boundary explicitly when registry or image size requires more time.

The `raw_exec` driver accepts `denied_envvars` in both driver and task
configuration, and supports selecting the task user on Windows. Review both
configuration levels when explaining why an environment variable is missing.

Executor failures in the `exec`, `raw_exec`, `java`, and `qemu` task drivers
report exit code `-1`. Treat this as an executor-launch failure rather than an
application exit status.

## QEMU machine configuration

Starting in 1.11.1, QEMU tasks accept `emulator` and `machine_type`, with
defaults of `qemu-system-x86_64` and `pc` (source batch `1.11-upgrade`). The
`kvm` accelerator no longer forces machine type `host`.

A `resources.cores` value supplies `-smp` only when the user has not supplied
a custom `-smp` flag. If topology differs from expectations, check both the
resources block and custom QEMU arguments.

Starting in 1.11.2, QEMU filesystem environment variables expose host file
paths instead of relative container paths such as `/alloc` and `/local`.
Update jobspecs and guest-launch arguments that consumed those variables; do
not translate the new values as though they were paths inside a container.

## Secrets plugin timeout

Secret-provider plugin execution times out after 60 seconds (source batch
`2.0.0`). Keep provider calls within the boundary and ensure failures surface
enough context to distinguish provider latency from authorization or
interpolation errors.

## Platform support

Nomad Enterprise 2.0 supports the Linux `ppc64le` CPU architecture. Validate
task artifacts, driver binaries, plugins, and container images for that
architecture independently; Nomad agent support does not make workload
dependencies portable automatically.
