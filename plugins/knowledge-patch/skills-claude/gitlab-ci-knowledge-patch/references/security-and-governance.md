# Security, Secrets, and Governance

## Store pipeline secrets

GitLab Secrets Manager enters open beta in 19.0 for Premium and Ultimate
customers on GitLab.com and GitLab Self-Managed. Project and group Owners can
store, retrieve, and reference secrets scoped to their project or group.

Pipeline access is restricted to jobs that explicitly request a secret. Keep
that request boundary visible in pipeline review rather than treating stored
secrets as globally available variables.

The service is governed by the beta support policy and might not be ready for
production workloads. Evaluate that status before relying on it for critical
secrets.

## Enforce scheduled pipelines from a policy project

Scheduled pipeline execution policies in 19.2 let a security policy project
define one schedule and enforce it across every project in scope. They do not
require a matching change to each project's `.gitlab-ci.yml`.

Each policy creates a separate pipeline independently of commit activity.
Configuration can cover:

- daily, weekly, or monthly cadence;
- time zone;
- distribution within an execution window; and
- target branch.

Use the central policy when the same compliance or security workload must run
consistently across many projects.

## Remediate vulnerable dependencies

Dependency scanning auto-remediation is beta in 19.2 on GitLab.com, GitLab
Self-Managed, and GitLab Dedicated. Once enabled, it monitors projects and
opens merge requests that move vulnerable dependencies to safe versions.

The default scope is patch and minor upgrades. Enabling the
credit-consuming Agentic Breaking Change Resolution also allows major
upgrades. For a failed update pipeline, it can:

1. analyze the dependency changelog;
2. inspect how the project uses the dependency;
3. commit compatibility fixes to the same merge request; and
4. rerun the pipeline until it passes.

Keep the major-upgrade behavior distinct from default auto-remediation when
estimating risk, review effort, and credit use.

