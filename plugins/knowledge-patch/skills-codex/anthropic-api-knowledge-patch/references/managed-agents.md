# Managed Agents and Administration

## Managed Agents surface

Claude Managed Agents provides API-managed agents, containers, built-in
tools, and server-sent event streaming. Core endpoints require:

```text
Anthropic-Beta: managed-agents-2026-04-01
```

Memory-list endpoints use a later beta header described below.

## Session startup and overrides

`POST /v1/sessions` accepts at most 50 `initial_events` of type
`user.message` or `user.define_outcome`. A non-empty list starts the agent loop
in the same call.

An agent with `type: "agent_with_overrides"` may replace the model, system
prompt, tools, MCP servers, or skills for only that session, without changing
the stored agent.

Put `effort` inside the agent's `model` object.

## Agent update concurrency

Supplying `version` in an agent update enables optimistic concurrency. A
version mismatch returns HTTP 409. Omitting `version` applies the update
unconditionally.

## Session and thread streams

`GET /v1/sessions/{session_id}/events/stream` accepts `event_deltas[]` to emit
`event_start` and `event_delta` previews before the complete `agent.message`.

The thread route `/threads/{thread_id}/stream` accepts the same parameter but
previews only the selected thread.

Session listings return both `prev_page` and `next_page`. Pass either cursor
back through `page`.

## Memory-list header cutover

For `GET /v1/memory_stores/{memory_store_id}/memories`, use:

```text
Anthropic-Beta: agent-memory-2026-07-22
```

The new contract:

- Uses stable server ordering.
- Ignores `order_by` and `order`.
- Accepts `depth` as 0, 1, or omitted.
- Requires `path_prefix` to end in `/` and match whole path segments.

This header replaces `managed-agents-2026-04-01` on memory endpoints. Sending
both returns HTTP 400. Old cursors cannot be reused. Explicit SDK beta lists
must replace the old header rather than append the new one. The old header
adopted the same list semantics on July 22.

## Memory, orchestration, and execution

Memory and multiagent orchestration are public beta under the standard
Managed Agents header. Dreams uses `dreaming-2026-04-21` to reorganize a
memory store from earlier sessions.

Agents may run in self-hosted sandboxes and alter MCP-server or tool
configuration during an active session.

Tool output over 100,000 characters spills into a sandbox file. The model
receives a truncated preview and the file path.

## Secrets

Vaults support environment-variable secrets and background refresh for
`mcp_oauth`. An `injection_location` controls substitution at egress into
request headers, the body, or both.

## Schedules and webhooks

Sessions may run on cron schedules.

Webhooks cover session, thread, vault, agent, deployment, deployment-run,
environment, and memory-store lifecycles. Events named
`session.thread_*` carry `session_thread_id`.

## Mid-conversation instructions

Fable 5, Mythos 5, and Opus 4.8 accept a `role: "system"` message in the
`messages` array immediately after a user turn, without a beta header. This
changes instructions while preserving the earlier prompt cache.

## Enterprise users and key lifetimes

The Enterprise user-management API manages members, invites, groups, and
roles. Group and custom-role operations require
`ce-user-management-2026-07-13`; member and invite operations do not.

An Admin key with `read:org_audit` may call every user-management `GET`
endpoint.

Console-created API and Admin API keys may now expire. Existing keys are
unchanged. Keys with a lifetime of at least seven days trigger a
pre-expiration email, and the Admin API reports the expiration.
