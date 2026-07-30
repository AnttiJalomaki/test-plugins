# Duo Agents and Flows

Use this reference when operating Duo from a terminal or pipeline, defining
reusable flows, handing work off from chat, or repairing CI/CD failures.

## Use Duo CLI from the terminal or CI/CD

Duo CLI is generally available through `glab` or as a standalone tool (since
19.2). It supports interactive chat and headless CI/CD execution.

It can use GitLab project, pipeline, and agent context and supports:

- shared sessions;
- model selection;
- tool approvals;
- MCP connections;
- slash commands;
- skills; and
- repository guidance from `AGENTS.md`.

Administrators of GitLab Self-Managed and GitLab Dedicated can disable Duo CLI.

## Define reusable custom flows

Duo custom flows are generally available as YAML-defined reusable workflows
(since 19.2). Manage them in a project or in the AI Catalog.

Custom flows support:

- multiple agents;
- human approval or feedback checkpoints;
- public or private visibility;
- validated configuration;
- service-account and composite-identity execution; and
- triggers based on mentions, assignments, pipeline events, and merge request
  lifecycle events.

Use project management for repository-owned workflows and catalog management
when the flow is intended for broader reuse.

## Start a foundational flow from Agentic Chat

Agentic Chat can hand a request to the Developer, Code Review, or Fix CI/CD
Pipeline foundational flow (since 19.2). The user approves the handoff in chat
and can follow the work either in the chat or under **AI** > **Sessions**.

## Use the CI Expert Agent

The CI Expert Agent is generally available for creating, debugging, and
optimizing pipelines from repository context (since 19.2).

When Orbit and its beta Knowledge Graph are enabled, the agent can also use
graph-based code intelligence for additional context. Treat Orbit and the
Knowledge Graph as separate enablement conditions.

## Get targeted fixes from the Fix CI/CD Pipeline Flow

The Fix CI/CD Pipeline Flow adds these production behaviors (since 19.2):

- It classifies failures before acting.
- When relevant files are already in a merge request diff, it returns fixes as
  code suggestions on that merge request.
- It follows child-pipeline failures through the complete hierarchy.
- It reads project-specific behavior from `AGENTS.md`.
- Its reasoning is collapsed by default in merge request comments.

Check the full pipeline hierarchy when a parent pipeline only reports the
downstream failure, and keep project-specific repair constraints in
`AGENTS.md`.
