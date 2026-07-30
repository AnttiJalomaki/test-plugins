# Pipeline Configuration and Credentials

## Select security scanner pipeline type

Since 18.0, `AST_ENABLE_MR_PIPELINES` selects whether AST security scanners
use a merge request pipeline or a branch pipeline while an MR is open:

```yaml
variables:
  AST_ENABLE_MR_PIPELINES: "true"
```

Stable security templates default to branch pipelines. Latest security
templates default to merge request pipelines. Set the variable explicitly
when template choice must not change the pipeline type.

The variable applies to every security scanning template except
`API-Discovery.gitlab-ci.yml`. API Discovery itself defaults to branch
pipelines in 18.0.

## Limit job-token permissions

Fine-grained job-token permissions are beta in 18.0 for projects in every tier
on GitLab.com and GitLab Self-Managed. They restrict a CI job token to
specific project resources rather than exposing the triggering user's full
permissions. Prefer these resource-specific permissions for least-privilege
jobs.

## Push to another project with a job token

Since 19.0, `CI_JOB_TOKEN` can push to a different project only when:

- the target project opts in;
- the user who started the pipeline has at least the Developer role in the
  target project; and
- the `allow_push_to_allowlisted_projects` feature flag is enabled.

The feature flag is disabled by default in 19.0. Treat a failed push as a
possible capability or role problem even when the target project is
allowlisted.

## Consume an individual array input

The CI/CD interpolation syntax supports the `[]` operator for direct array
element access since 19.0:

```yaml
spec:
  inputs:
    targets:
      type: array
      default: [staging, production]
---
show-first-target:
  script:
    - echo "$[[ inputs.targets[0] ]]"
```

This avoids an extra step whose only purpose is to unpack an array before the
job can use a selected element.

## Accept multiple Run pipeline selections

For an array input with options, the Run pipeline UI can select multiple
values. GitLab combines them into a single array, such as:

```json
["option1", "option2"]
```

Define the input as an array and make downstream jobs consume an array rather
than assuming the UI supplies one scalar value.

## Control merge-train concurrency

Premium and Ultimate customers on GitLab Self-Managed and GitLab Dedicated
can configure merge-train pipeline concurrency since 19.0. The setting
replaces the former fixed maximum of 20 parallel pipelines and can be applied
per project or across the instance.

Use a limit of `1` to process merge requests serially against a clean target
branch. Higher limits trade that isolation for parallelism.

