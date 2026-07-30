# Jobs and Scheduling

## Submission compatibility

Starting in 1.11.0, `system` and `sysbatch` jobs containing a `reschedule`
block fail submission. Remove the block; earlier releases silently ignored it.
This upgrade check is from batch `1.11-upgrade`.

A task cannot be named `alloc`, because that name breaks inter-task filesystem
isolation. Rename the task before submission (batch `1.11.0`).

Previously deprecated task-group disconnect fields have no effect. Use the
`disconnect` block introduced in Nomad 1.8 (batch `1.10.0`).

## System deployments and group lifecycle

System jobs support deployments controlled through the job's `update`
configuration. Canary and blue/green strategies are available, and deployment
status is visible in the web UI and through `nomad deployment` commands.

For service and batch jobs, changing a task group to `count = 0` behaves like
removing the group: Nomad stops every non-terminal allocation for that group.

Both behaviors are from batch `1.11.0`.

## Job updates and interpolation

Affinity and spread changes are no longer destructive. During in-place
updates, Nomad-native services interpolate correctly. Task-level services,
checks, and identities no longer interpolate jobspec values belonging to other
tasks in the same group.

Use `-preserve-resources` on the job-update CLI to retain the existing resource
block while updating the job.

The interpolation behavior is from batch `1.10.0`; resource preservation is
from batch `1.11.0`.

## Job maximum count

The `job_max_count` server option limits the sum of all task-group `count`
values when a job is submitted or scaled:

```hcl
server {
  job_max_count = 100000
}
```

It defaults to `50000`. Changing the value does not affect jobs that already
exist. This option is from batch `1.11-upgrade`.

## Placement and evaluation diagnostics

`nomad eval status` shows related evaluations, placed allocations, plan
annotations, failed placements, and preemptions. More fields are visible
without `-verbose`, and reconciler annotations describe the intended plan
before node-feasibility checks.

`nomad alloc status -verbose` adds evaluated-node and rejected-node counts plus
node scores. The Go API's `Evaluations.Info` populates `RelatedEvals`.

These diagnostic additions are from batch `1.11.0`.

## Allocation commands

The following commands accept `-group` to select a task group:

```shell
nomad alloc exec -group=<group> <allocation> <command>
nomad alloc logs -group=<group> <allocation>
nomad alloc fs -group=<group> <allocation>
```

Group selection is from batch `1.10.0`.

## Consul Connect networking

Consul Connect permits `cni/*` network modes. The combination is explicitly
marked use-at-your-own-risk (batch `1.11.0`).

## Scheduling resource calculations

Nomad calculates disk available for scheduling as:

```text
totalBytes - client.reserved.disk
```

It no longer uses free disk space and removes the
`unique.storage.bytesfree` attribute. Reserve at least the disk capacity used
by the host operating system. This placement change is from batch
`1.11-upgrade`.
