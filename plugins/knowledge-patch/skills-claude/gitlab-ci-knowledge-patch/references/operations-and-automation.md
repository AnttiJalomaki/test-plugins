# Operations, Observability, and Automation

## Inspect project pipeline analytics

The redesigned project CI/CD analytics view is limited availability on GitLab
Dedicated in 18.0. It exposes pipeline performance trends and reliability
metrics in the project UI. Do not assume the same availability on other
deployment types.

## Find CI/CD Catalog component consumers

Since 19.0, an Ultimate customer can open a catalog resource's usage details
to see:

- which projects consume each component;
- which version each consumer selected; and
- whether that version is current.

Outdated consumers appear first. The capability is available on GitLab.com,
GitLab Self-Managed, and GitLab Dedicated.

## Export Runner telemetry

GitLab Runner 19.0 adds instrumentation feature negotiation and an OTLP export
client. Its first emitted trace data includes a `job_execution` span. Configure
an OTLP destination when job-level execution traces should feed the existing
observability system.

## Tune Runner preparation

The Runner prepare-stage timeout is configurable in 19.0. Set it in Runner
configuration when environment preparation legitimately needs a different
timeout.

## Work from the terminal or CI/CD

GitLab Duo CLI is generally available in 19.2 through `glab` or as a
standalone tool. It offers interactive chat for terminal work and a headless
mode suitable for CI/CD.

It can use GitLab project, pipeline, and agent context and supports:

- shared sessions;
- model selection and tool approvals;
- MCP connections;
- slash commands and skills; and
- repository guidance from `AGENTS.md`.

Administrators of GitLab Self-Managed and GitLab Dedicated can disable the
tool.

## Define reusable custom flows

GitLab Duo custom flows are generally available in 19.2. They are YAML-defined
workflows managed in a project or the AI Catalog and can:

- coordinate multiple agents;
- pause for human approval or feedback;
- use public or private visibility;
- validate their configuration;
- execute with service-account and composite identities; and
- start from mentions, assignments, pipeline events, or merge request
  lifecycle events.

Choose a custom flow when the automation needs a reusable, governed sequence
rather than a single ad hoc interaction.

## Hand work off from Agentic Chat

In 19.2, Agentic Chat can transfer a request to the Developer, Code Review, or
Fix CI/CD Pipeline foundational flow. The user approves the handoff in chat
and can follow the resulting work there or under **AI** > **Sessions**.

## Create and optimize pipelines with the CI Expert Agent

The CI Expert Agent is generally available in 19.2 for creating, debugging,
and optimizing pipelines from repository context. When Orbit and its beta
Knowledge Graph are enabled, the agent can also draw on graph-based code
intelligence for more contextual recommendations.

## Classify and fix pipeline failures

The Fix CI/CD Pipeline Flow in 19.2 classifies a failure before taking action.
When the relevant files already appear in a merge request diff, it returns
fixes as code suggestions on that merge request.

The flow also:

- follows child-pipeline failures through the complete hierarchy;
- reads project-specific behavior from `AGENTS.md`; and
- collapses its reasoning in merge request comments by default.

Provide repository guidance in `AGENTS.md` when project conventions should
affect the proposed fix, and inspect the full pipeline hierarchy when the
visible failure originated in a child pipeline.

