# Security, Secrets, and Job Tokens

Use this reference when reducing CI token permissions, enabling cross-project
pushes, storing pipeline secrets, or automating vulnerable dependency updates.

## Fine-grained CI job-token permissions

Projects on GitLab.com and GitLab Self-Managed can restrict a CI job token to
specific project resources instead of granting the full permissions of the user
who triggered the pipeline (beta since 18.0).

The beta is available in all tiers. Use it to give each CI/CD job only the
project-resource access that it needs.

## Push to another project with `CI_JOB_TOKEN`

GitLab 19.0 can allow a job token to push to a different project when all of
these conditions hold:

1. The target project opts in.
2. The user who started the pipeline has at least the Developer role in the
   target project.
3. The `allow_push_to_allowlisted_projects` feature flag is enabled.

The feature flag is disabled by default. Account for both the target-project
allowlist decision and the triggering user's role when diagnosing a denied
push.

## Store and request secrets

GitLab Secrets Manager is in open beta for Premium and Ultimate customers on
GitLab.com and GitLab Self-Managed (since 19.0).

- Project and group Owners can store and retrieve secrets scoped to their
  project or group.
- Access can be limited to pipeline jobs that explicitly request a secret.
- The service remains subject to the beta support policy and might not be
  production-ready.

Treat the explicit job request as part of the secret's access boundary; storing
a secret at project or group scope does not make it implicitly available to
every job.

## Automatically remediate vulnerable dependencies

Dependency scanning auto-remediation is available in beta on GitLab.com,
GitLab Self-Managed, and GitLab Dedicated (since 19.2). When enabled, it
monitors projects and opens merge requests that update vulnerable dependencies
to safe versions.

By default, remediation is limited to patch and minor upgrades. Enabling the
credit-consuming Agentic Breaking Change Resolution also permits major
upgrades. For a failed update pipeline, that capability can:

- analyze the dependency changelog;
- inspect code usage;
- commit compatibility fixes to the same merge request; and
- rerun the pipeline until it passes.

Choose whether major upgrades and credit consumption are acceptable before
enabling the additional capability.
