# Platforms and Release Lifecycle

Use this reference for retirement planning, capability discovery, authentication,
hosted-platform differences, and administrative migrations.

## Lifecycle states and usage audits

Legacy models no longer receive updates but have no retirement date yet.
Deprecated models continue working until their assigned retirement; requests to
retired models fail. Publicly released models receive at least 60 days' notice
before retirement.

Use the Console usage export to obtain a CSV grouped by API key and model, then
locate every caller of an ID before migrating it.

## Retirement schedule

On Anthropic-operated platforms:

- `claude-mythos-preview` is deprecated in favor of `claude-mythos-5`.
- `claude-opus-4-1-20250805` retires August 5, 2026; move to
  `claude-opus-4-8` first.
- The listed Claude 4.0, 3.x, 2.x, 1.x, and Instant IDs are already retired.
- Earliest tentative retirements are November 24, 2026 for Opus 4.5;
  September 29, 2026 for Sonnet 4.5; and October 15, 2026 for Haiku 4.5.
- Earliest tentative retirements are February 5, April 16, May 28, and July 24,
  2027 for Opus 4.6, 4.7, 4.8, and 5 respectively.
- Earliest tentative retirements are February 17 and June 30, 2027 for Sonnet
  4.6 and 5, and June 9, 2027 for Fable 5.

Treat tentative dates as planning floors and recheck the current lifecycle
record before a production cutover.

## Removed and retiring request behavior

Although SDK request types still expose `temperature`, `top_p`, and `top_k`,
non-default values return HTTP 400 on Opus 4.7 and later and on Mythos Preview.
Omit these fields and steer with effort.

Legacy Workbench access ends August 17, 2026. Its saved prompts, variables, and
evals do not transfer to the updated experience, so export required data first.
The experimental endpoints below retire the same day and then return errors:

- `/v1/experimental/generate_prompt`
- `/v1/experimental/improve_prompt`
- `/v1/experimental/templatize_prompt`

`speed: "fast"` on Opus 4.7 now returns an error. On Opus 4.6 it silently runs
at standard speed and standard pricing. Check `usage.speed` rather than assuming
the requested mode was honored, and move fast workloads to a supported target.

## Context and output-limit cutovers

`context-1m-2025-08-07` has no effect on Sonnet 4 or 4.5; requests to those
models above their standard 200k context now fail. Opus 4.6 and Sonnet 4.6
instead provide 1M context at standard pricing without a beta header, use
ordinary account rate limits at all context lengths, and accept up to 600 images
or PDF pages in a 1M-context request.

Discover limits dynamically with `GET /v1/models` or
`GET /v1/models/{model_id}`. Their records expose `max_input_tokens`,
`max_tokens`, and `capabilities`. Opus 4.6 and Sonnet 4.6 can increase the
single-turn output limit to 300k with `output-300k-2026-03-24`.

## Workload identity and inference location

Workload Identity Federation is GA. Configure OIDC issuers and federation rules
in the Console, then let an SDK exchange and refresh short-lived credentials
instead of distributing static API keys.

For models released after February 1, 2026, `inference_geo` can request US-only
inference at 1.1x pricing.

## Two distinct AWS-hosted surfaces

Claude Platform on AWS runs Anthropic-managed infrastructure with AWS billing
and IAM. Its native AWS endpoints expose Messages, Files, Message Batches,
Managed Agents, Agent Skills, code execution, and tool use.

Amazon Bedrock is separate AWS-managed infrastructure. Its
`/anthropic/v1/messages` endpoint accepts the first-party Messages request
shape. Opus 4.7 and Haiku 4.5 are self-serve there through global and regional
endpoints. Do not infer one surface's feature support from the other.

## MCP tunnel migration

Tunnel management moved from Admin API `/v1/organizations/tunnels` to Claude
API `/v1/tunnels`. The new route requires `mcp-tunnels-2026-06-22` and the
`workspace:manage_tunnels` WIF scope. The old route remains only for a migration
window.

## Mid-conversation system messages

Fable 5, Mythos 5, and Opus 4.8 accept a `role: "system"` message in the
`messages` array immediately after a user turn, without a beta header. This can
change instructions while preserving the earlier prompt cache. Other
model-specific behavior is listed in
[Models and Migrations](models-and-migrations.md).

## Enterprise users and expiring keys

The Enterprise user-management API manages members, invites, groups, and roles.
Group and custom-role operations require `ce-user-management-2026-07-13`;
member and invite operations do not. An Admin key with `read:org_audit` may call
all user-management `GET` routes.

Console-created API and Admin API keys can expire. Existing keys are unchanged.
Keys with lifetimes of at least seven days trigger a pre-expiration email, and
the Admin API reports the expiration time.
