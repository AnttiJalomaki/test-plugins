---
name: gitlab-ci-knowledge-patch
description: GitLab CI/CD
version: 19.2
license: MIT
metadata:
  author: Nevaberry
---


# GitLab CI/CD Knowledge Patch

Use this skill when authoring, reviewing, debugging, or upgrading GitLab
pipelines and their supporting GitLab installation. It is especially useful
when work involves pipeline inputs, job-token boundaries, security scanning,
merge trains, Runner telemetry, GitLab Duo automation, or a GitLab 19 upgrade.

Start by identifying the deployment type, exact GitLab and Runner versions,
license tier, and relevant feature flags. Several capabilities differ between
GitLab.com, GitLab Self-Managed, and GitLab Dedicated, and some are beta,
limited-availability, opt-in, or disabled by default.

## Reference index

| Reference | Topics |
|---|---|
| [references/pipeline-configuration.md](references/pipeline-configuration.md) | AST pipeline selection, typed inputs, merge trains, job-token permissions and cross-project pushes |
| [references/security-and-governance.md](references/security-and-governance.md) | Secrets Manager, scheduled execution policies, dependency remediation |
| [references/operations-and-automation.md](references/operations-and-automation.md) | Analytics, catalog usage, Runner telemetry and timeouts, terminal tools, agents and flows |
| [references/platform-upgrades.md](references/platform-upgrades.md) | Required upgrade stops, database and registry migrations, removed services, Helm, operating systems, Geo and RPM cleanup |

## Upgrade blockers first

### Follow every required stop

For GitLab 19 upgrades, include required stops `19.2`, `19.5`, `19.8`, and
`19.11` in the upgrade plan. Read the notes for every release between the
current and target versions, including installation-method-specific notes.

### Do not target 19.2.0 for direct Linux package upgrades

A direct upgrade to 19.2.0 can erase locally configured AI Gateway and Duo
Agent Platform service URLs and reset related settings. Target 19.2.1 or
later. If the reset already happened, restore the endpoints under
**Admin area** > **GitLab Duo** > **Configuration** > **Service endpoints**.

### Upgrade PostgreSQL before GitLab

PostgreSQL 17 is required before installing GitLab 19, regardless of
installation method. Upgrade packaged PostgreSQL 16 or the external database
first.

### Prepare registry configuration

Existing Linux package and self-compiled installations without an explicit
registry metadata database setting enter `prefer` mode. On 19.0.0 and 19.0.1,
this can break `/gitlab/v1/` routes even while `/v2/` pushes and pulls work.
Use the documented temporary disable-and-restart recovery, then upgrade to
19.0.2 or later.

The legacy registry `s3` driver is removed and aliases to `s3_v2`. For
non-AWS S3-compatible storage, use a complete URI for `regionendpoint` and set
`checksum_disabled` if the backend rejects enhanced upload checksums. Deletion
still sends CRC32, so the backend itself must support it.

### Externalize removed dependencies

Before upgrading affected installations:

- replace external Redis 6 with Redis 7.0 or later, or Valkey 7.2;
- move bundled Mattermost users to standalone Mattermost and remove every
  `mattermost[...]` key before reconfigure;
- deploy Spamcheck separately;
- replace the Helm chart's removed bundled PostgreSQL, Redis, and MinIO
  services with external services;
- prepare the Helm chart's Gateway API and Envoy Gateway default, or
  temporarily re-enable bundled NGINX Ingress.

See [references/platform-upgrades.md](references/platform-upgrades.md) for the
complete upgrade checklist, including Ubuntu, SUSE, Geo, and RPM cases.

## Pipeline behavior quick reference

### Choose AST merge request or branch pipelines explicitly

Set `AST_ENABLE_MR_PIPELINES: "true"` to choose merge request pipelines when
an MR is open:

```yaml
variables:
  AST_ENABLE_MR_PIPELINES: "true"
```

Stable security templates otherwise default to branch pipelines, while Latest
templates default to merge request pipelines. The option covers all security
scanning templates except `API-Discovery.gitlab-ci.yml`; API Discovery
defaults to branch pipelines.

### Index typed array inputs directly

Use `[]` during CI/CD input interpolation instead of adding a processing step:

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

For array inputs with options, the Run pipeline UI supports multiple
selections and supplies the selected values as one array.

### Treat job tokens as scoped credentials

Fine-grained job-token permissions can limit a token to selected project
resources instead of inheriting all permissions of the triggering user. The
capability is beta and available in all tiers on GitLab.com and GitLab
Self-Managed.

Cross-project pushes require all of the following:

1. the target project opts in;
2. the user who started the pipeline has at least Developer access there;
3. `allow_push_to_allowlisted_projects` is enabled where required.

That feature flag is disabled by default. Do not assume that an allowlist alone
makes a cross-project push available.

### Set merge-train concurrency deliberately

Self-Managed and Dedicated Premium or Ultimate installations can configure a
per-project or instance-wide merge-train pipeline limit instead of the former
fixed maximum of 20. Set the limit to `1` when merge requests must be tested
one at a time against a clean target branch.

## Security and policy quick reference

### Scope secrets to requesting jobs

GitLab Secrets Manager is open beta for Premium and Ultimate on GitLab.com and
GitLab Self-Managed. Project and group Owners can store and reference secrets
at project or group scope, while access is limited to jobs that explicitly
request them. Apply the beta support policy and do not assume production
readiness.

### Schedule security pipelines centrally

Scheduled pipeline execution policies define a schedule once in a security
policy project and enforce it across all in-scope projects without editing
each `.gitlab-ci.yml`. Each policy starts a separate pipeline independently of
commit activity and can control cadence, time zone, execution-window
distribution, and target branch.

### Bound dependency remediation

Dependency scanning auto-remediation is beta. By default, it opens merge
requests for safe patch or minor updates. The credit-consuming Agentic
Breaking Change Resolution option also permits major upgrades and can inspect
the failed pipeline, dependency changelog, and code usage, commit compatibility
fixes, and rerun the pipeline until it passes.

## Runner and project operations

Runner 19.0 can negotiate instrumentation, export through OTLP, and emit its
first `job_execution` span. Configure the prepare-stage timeout in Runner
configuration when the default does not fit the environment.

On GitLab Dedicated, the redesigned project CI/CD analytics view is limited
availability and exposes pipeline performance and reliability trends.

Ultimate users can inspect catalog component consumers, their selected
versions, and whether those versions are current. Outdated consumers sort
first, which makes the view useful for component migration work.

## Assisted workflow quick reference

GitLab Duo CLI is available through `glab` or as a standalone tool. Use
interactive chat for exploratory work and headless mode for CI/CD. It can use
project, pipeline, and agent context, and supports shared sessions, model
selection, tool approvals, MCP connections, slash commands, skills, and
`AGENTS.md`. Self-Managed and Dedicated administrators can disable it.

Custom flows are reusable YAML workflows stored in a project or the AI
Catalog. They can coordinate multiple agents, require human approval or
feedback, use public or private visibility, validate configuration, run under
service-account and composite identities, and react to repository and pipeline
events.

Agentic Chat can hand work to the Developer, Code Review, or Fix CI/CD
Pipeline foundational flow after user approval. Track the handoff in chat or
under **AI** > **Sessions**.

The CI Expert Agent can create, debug, and optimize pipelines from repository
context. When Orbit and its beta Knowledge Graph are enabled, it can use
graph-based code intelligence.

The Fix CI/CD Pipeline Flow classifies failures before changing code. It can
return merge request code suggestions when affected files are already in the
diff, follow child-pipeline failures through the hierarchy, read project
behavior from `AGENTS.md`, and collapse its reasoning in merge request
comments by default.

## Review checklist

Before changing a pipeline:

- identify GitLab, Runner, tier, deployment, and feature-flag constraints;
- check whether a feature is beta, limited availability, opt-in, or generally
  available;
- preserve the distinction between branch and merge request pipeline
  behavior;
- verify job-token access on both the source and target projects;
- test typed input interpolation with the values produced by the UI;
- set merge-train concurrency according to desired target-branch isolation;
- confirm secrets and scheduled policies reach only the intended jobs and
  projects;
- inspect parent and child pipeline failures before proposing a fix.

Before upgrading an installation:

- visit each required stop and read intervening release notes;
- upgrade PostgreSQL and external Redis or Valkey first;
- inventory registry storage and metadata database settings;
- externalize services removed from packages, charts, or the Operator;
- verify operating-system and installation-method support;
- plan recovery for Geo replication and known package cleanup cases;
- back up service endpoint configuration before a direct package upgrade.

