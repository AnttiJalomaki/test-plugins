# Managed Agents

Use this reference for API-managed agents, sessions, event streams, memory,
vaults, schedules, and webhooks.

## Core beta contract

Claude Managed Agents provides API-managed agents, containers, built-in tools,
and server-sent event streaming. The core surface uses
`managed-agents-2026-04-01`; memory endpoints use the later header described
below.

## Starting sessions and applying overrides

`POST /v1/sessions` accepts up to 50 `initial_events`. Each must be
`user.message` or `user.define_outcome`. A non-empty list starts the agent loop
within the same request.

For a single-session customization, provide an `agent` with
`type: "agent_with_overrides"`. It can replace the model, system prompt, tools,
MCP servers, or skills without modifying the stored agent.

Place `effort` inside the agent's `model` object. When updating an agent, include
`version` for optimistic concurrency: a mismatch returns HTTP 409. Omitting
`version` applies the update unconditionally.

## Session and thread event streams

`GET /v1/sessions/{session_id}/events/stream` accepts `event_deltas[]` and can
emit `event_start` and `event_delta` previews before the completed
`agent.message`.

The thread route `/threads/{thread_id}/stream` accepts the same parameter but
previews only the selected thread. Session list responses include `prev_page`
as well as `next_page`; pass either cursor back through `page`.

## Memory-list header cutover

With `agent-memory-2026-07-22`,
`GET /v1/memory_stores/{memory_store_id}/memories`:

- uses stable server ordering and ignores `order_by` and `order`;
- accepts `depth` only as 0, 1, or omitted; and
- requires `path_prefix` to end in `/` and match complete path segments.

This header replaces `managed-agents-2026-04-01` on memory endpoints. Sending
both returns HTTP 400. Old cursors cannot be reused. Explicit SDK beta lists
must replace the old value, not append the new one. The old header adopted the
same list semantics on July 22.

## Memory, orchestration, and execution

Memory and multiagent orchestration are public beta under the standard Managed
Agents header. Dreams uses `dreaming-2026-04-21` to reorganize a memory store
using prior sessions.

Agents can run in self-hosted sandboxes and can change MCP-server or tool
configuration during an active session. Tool output over 100,000 characters
spills to a sandbox file; the model receives a truncated preview and the file
path.

## Vaults, schedules, and webhooks

Vaults support environment-variable secrets and background refresh for
`mcp_oauth`. Set `injection_location` to substitute credentials at egress into
request headers, the body, or both.

Sessions can run on cron schedules. Webhooks cover session, thread, vault,
agent, deployment, deployment-run, environment, and memory-store lifecycles.
`session.thread_*` events include `session_thread_id`.

Managed Agents request pools are documented in
[Prompt Caching and Rate Limits](caching-and-rate-limits.md).
