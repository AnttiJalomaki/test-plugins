# Jobs, Scheduling, and Deployments

Use this reference for jobspec validation, system-job rollouts, scale limits,
update behavior, interpolation, and placement-sensitive networking.

## Rejected and reserved jobspec content

`system` and `sysbatch` jobs fail submission when they contain a `reschedule`
block. Earlier releases silently ignored it. Remove the block rather than
expecting system-job rescheduling semantics (source batch `1.11-upgrade`).

Tasks may not be named `alloc`, because that name breaks inter-task filesystem
isolation. Rename the task before submission (source batch `1.11.0`).

Previously deprecated task-group disconnect fields have no effect. Replace
them with the `disconnect` block.

## Per-job allocation limit

The `job_max_count` server option defaults to `50000`. At job submission and
scale time, it limits the sum of all task-group `count` values:

```hcl
server {
  job_max_count = 100000
}
```

Changing the option does not affect existing jobs. Account for this distinction
when a previously accepted job is later updated or scaled.

## System-job deployments

System jobs support deployments and controlled rollouts through their `update`
configuration, including canary and blue/green strategies. Deployment status
is available in the web UI and through `nomad deployment` commands.

This does not restore the rejected `reschedule` block: use deployment and
update behavior for rollout control, and remove reschedule configuration.

## Updates, resource preservation, and zero counts

Affinity and spread changes are no longer classified as destructive. During
in-place updates:

- Nomad-native services interpolate correctly.
- Task-level services, checks, and identities do not interpolate jobspec
  values from other tasks in the group.

Keep task-local interpolation dependencies explicit instead of depending on
cross-task values (source batch `1.10.0`).

The job-update CLI accepts `-preserve-resources` to retain the existing
resource block while updating a job. Use it only when retaining the deployed
resource values is intentional.

For service and batch jobs, changing a task group to `count = 0` behaves like
removing the group and stops every non-terminal allocation. Treat this as a
destructive workload transition even though the group remains in the jobspec.

## Consul Connect and CNI networking

Consul Connect permits `cni/*` network modes. The combination is explicitly
marked use-at-your-own-risk. Validate CNI setup, service connectivity,
allocation lifecycle, and failure recovery before adopting it for production
workloads.

## Scheduling diagnostics

Use the expanded evaluation and allocation diagnostics when placement fails:

- `nomad eval status` shows related evaluations, placed allocations, plan
  annotations, failed placements, and preemptions, with more information
  available without `-verbose`.
- Reconciler annotations describe the intended plan before node-feasibility
  checks.
- `nomad alloc status -verbose` reports evaluated and rejected node counts and
  node scores.

These diagnostics help separate job constraints, scheduler intent, and node
feasibility before changing a jobspec.
