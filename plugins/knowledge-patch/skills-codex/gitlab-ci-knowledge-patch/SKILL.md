---
name: gitlab-ci-knowledge-patch
description: GitLab CI/CD
version: 19.2
license: MIT
metadata:
  author: Nevaberry
---


# GitLab CI/CD Knowledge Patch

Use this skill when writing or reviewing GitLab pipelines, configuring Runner,
planning a GitLab upgrade, or adopting GitLab's security and automation
features. Start with the breaking upgrade guidance, then load the topic
reference that matches the task.

## Reference index

| Reference | Topics |
| --- | --- |
| [upgrade-planning.md](references/upgrade-planning.md) | Required upgrade stops, database and registry changes, operating-system migrations, removed bundled services, Helm changes, and upgrade defect recovery |
| [pipeline-configuration.md](references/pipeline-configuration.md) | Security-scanner pipeline selection, typed array inputs, merge-train concurrency, and centrally scheduled pipeline policies |
| [security-secrets-and-job-tokens.md](references/security-secrets-and-job-tokens.md) | Fine-grained job-token permissions, cross-project pushes, Secrets Manager, and dependency auto-remediation |
| [runner-catalog-and-analytics.md](references/runner-catalog-and-analytics.md) | Runner OTLP telemetry and prepare timeout, CI/CD analytics, and CI/CD Catalog usage |
| [duo-agents-and-flows.md](references/duo-agents-and-flows.md) | Duo CLI, reusable custom flows, Agentic Chat handoffs, CI Expert Agent, and pipeline-fix behavior |

## Breaking changes and upgrade hazards

### Plan every required stop

- Route GitLab 19 upgrades through `19.2`, `19.5`, `19.8`, and `19.11`.
- Review every intermediate release note, including notes specific to the
  installation method.
- Do not upgrade a direct Linux package installation to `19.2.0`; use `19.2.1`
  or later to avoid losing local Duo service endpoint settings.
- Upgrade PostgreSQL to 17 before installing GitLab 19, regardless of the
  installation method.

### Prepare the container registry

- Existing Linux package and self-compiled installations should explicitly
  evaluate the registry metadata database setting before GitLab 19.
- If `19.0.0` or `19.0.1` returns HTTP 500 from `/gitlab/v1/`, temporarily
  disable the metadata database, reconfigure, restart the registry, and remove
  the override after reaching `19.0.2` or later.
- Replace legacy `s3` storage configuration with `s3_v2`. Use a complete URI
  for a non-AWS `regionendpoint`.
- Set `checksum_disabled` only for backends that reject enhanced upload
  checksums. Deletion still sends CRC32 and requires backend support.

### Replace removed platforms and bundled services

- Move Ubuntu 20.04 and supported-SUSE-package installations before GitLab 19.
- Replace external Redis 6 with Redis 7.0 or later or Valkey 7.2.
- Externalize Mattermost and remove every `mattermost[...]` setting before
  reconfiguration.
- Externalize Spamcheck; no data migration is required.
- For Helm and Operator deployments, provision external PostgreSQL, Redis, and
  object storage before upgrading.
- Account for the Helm chart's move from bundled NGINX Ingress to Gateway API
  with Envoy Gateway.

### Recover known upgrade defects

- Restore local AI Gateway and Duo Agent Platform endpoints in the Admin area
  if an upgrade to `19.2.0` cleared them.
- Upgrade both Geo sites to `19.0.2` or later if OCI image-index tags were
  omitted, then wait for verification or manually resync affected container
  repositories.
- On affected RPM upgrades, remove the documented orphaned agent directories
  only after reaching a fixed release.

Open [upgrade-planning.md](references/upgrade-planning.md) before executing an
upgrade; it contains the exact settings, affected releases, paths, and recovery
conditions.

## High-value pipeline configuration

### Choose merge-request or branch security scanning

Set `AST_ENABLE_MR_PIPELINES` explicitly when the desired security-scanner
pipeline type differs from the selected template's default:

```yaml
variables:
  AST_ENABLE_MR_PIPELINES: "true"
```

Stable security templates default to branch pipelines, while Latest templates
default to merge request pipelines. The setting covers the security scanning
templates except API Discovery.

### Use individual array input values

Index an array input directly during interpolation:

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

The Run pipeline UI can also collect multiple choices for an array input whose
allowed options are declared. The selected values arrive as one array.

### Control merge-train pressure

- Set merge-train concurrency per project or across an instance where the
  feature is available.
- Use a limit of `1` when merge requests must be tested one at a time against a
  clean target branch.
- Treat this as a replacement for the former fixed maximum of 20 parallel
  merge-train pipelines on supported deployments.

### Enforce scheduled pipelines centrally

Use a scheduled pipeline execution policy when the same schedule must apply to
all projects in a security-policy scope without editing each
`.gitlab-ci.yml`. Configure cadence, time zone, execution-window distribution,
and target branches in the policy.

## Security, secrets, and token boundaries

### Minimize job-token access

- Prefer fine-grained CI job-token permissions where the beta is enabled.
- Limit a token to the project resources a job actually needs instead of
  inheriting the triggering user's broad permissions.
- For cross-project pushes, require the target project's opt-in and verify that
  the user who started the pipeline has at least the Developer role there.
- Remember that cross-project push support has an additional disabled-by-default
  feature flag.

### Request secrets explicitly

GitLab Secrets Manager can scope secrets to a project or group and restrict
their use to pipeline jobs that explicitly request them. Treat the open beta as
subject to beta support expectations rather than assuming production readiness.

### Bound automated dependency updates

- Dependency scanning auto-remediation opens merge requests for vulnerable
  dependencies and defaults to safe patch or minor releases.
- Major upgrades require the credit-consuming breaking-change capability.
- That capability can inspect a failed pipeline, dependency changelog, and code
  usage; commit compatibility fixes; and rerun the same merge request pipeline.

## Runner, catalog, and analytics

### Export Runner telemetry

Runner can negotiate instrumentation and export OTLP telemetry. Its initial
trace signal is the `job_execution` span. Configure the collector side with
that initial scope in mind.

### Bound the prepare stage

Use the Runner configuration's prepare-stage timeout when environment
preparation needs a deliberate upper bound. Keep this separate from job script
timeouts.

### Inspect pipeline and component adoption

- The redesigned project CI/CD analytics view on GitLab Dedicated exposes
  pipeline performance trends and reliability metrics under limited
  availability.
- CI/CD Catalog usage details show which projects consume a component, the
  selected versions, and whether those versions are current. Outdated consumers
  are listed first.

## Duo agents and reusable flows

### Choose an interaction surface

- Use Duo CLI through `glab` or the standalone tool for interactive terminal
  work or headless CI/CD execution.
- Use custom flows for reusable YAML-defined workflows with triggers,
  identities, visibility controls, validation, and human checkpoints.
- Use Agentic Chat to hand an approved request to the Developer, Code Review,
  or Fix CI/CD Pipeline foundational flow.

### Apply repository-specific pipeline repair

- The CI Expert Agent can create, debug, and optimize pipelines using repository
  context.
- The Fix CI/CD Pipeline Flow classifies failures before changing code.
- When relevant files are already in a merge request diff, it can return fixes
  as merge request code suggestions.
- It follows child-pipeline failures through the full hierarchy and accepts
  project-specific instructions from `AGENTS.md`.

## Task workflow

1. Identify the GitLab version, deployment type, tier, and whether a feature is
   generally available, beta, or limited availability.
2. For upgrades, inventory the installation method, database, registry storage,
   operating system, external services, Geo topology, and Helm or Operator
   dependencies.
3. For pipeline changes, identify the pipeline source, template family, token
   trust boundary, input types, and any instance-wide policy.
4. Open every reference relevant to the change; availability and recovery
   details differ by deployment type, tier, and point release.
5. Preserve explicit opt-ins and feature flags in implementation plans. Do not
   treat a documented capability as enabled by default.
6. Validate generated pipeline behavior and upgrade recovery steps in the
   project's own environment before rollout.
