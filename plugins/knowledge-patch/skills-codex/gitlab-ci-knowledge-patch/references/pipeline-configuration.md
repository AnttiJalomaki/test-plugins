# Pipeline Configuration

Use this reference for pipeline-source selection, typed inputs, merge trains,
and centrally enforced schedules.

## Select security-scanner pipeline behavior

GitLab 18.0 adds `AST_ENABLE_MR_PIPELINES` to choose whether AST security
scanners use merge request or branch pipelines when a merge request is open.

```yaml
variables:
  AST_ENABLE_MR_PIPELINES: "true"
```

The defaults differ by template family:

- Stable security templates default to branch pipelines.
- Latest security templates default to merge request pipelines.
- The variable applies to every security scanning template except
  `API-Discovery.gitlab-ci.yml`.
- API Discovery itself defaults to branch pipelines in 18.0.

Override the template behavior explicitly when pipeline type matters to the
project.

## Index CI/CD array inputs

CI/CD input interpolation supports the `[]` array index operator (since 19.0).
A job can consume one element without an intermediate processing step:

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

## Accept multiple choices in the Run pipeline UI

For an array input with declared options, the Run pipeline UI allows multiple
selections (since 19.0). GitLab combines the selections into one array, such as
`["option1","option2"]`, so a single run can operate on several targets.

Use array interpolation or pass the array to a job that understands its typed
value; do not add string-splitting solely to recover the selected elements.

## Configure merge-train concurrency

Premium and Ultimate customers on GitLab Self-Managed and GitLab Dedicated can
configure a merge-train pipeline limit per project or across the instance
(since 19.0). This replaces the former fixed maximum of 20 parallel pipelines.

A limit of `1` processes merge requests one at a time against a clean target
branch. Select it when serialized validation matters more than train
throughput.

## Schedule pipelines through security policy

Scheduled pipeline execution policies can define one schedule in a security
policy project and enforce it across all projects in scope (since 19.2). The
target projects do not need corresponding changes to `.gitlab-ci.yml`.

Each policy starts a separate pipeline independently of commit activity. A
policy can define:

- daily, weekly, or monthly cadence;
- time zone;
- distribution within an execution window; and
- target branches.

Use policy scope to choose the affected projects and keep centrally required
schedules out of project-local pipeline files.
