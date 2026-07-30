# Platforms, Discovery, and Limits

## Discover model limits

`GET /v1/models` and `GET /v1/models/{model_id}` return
`max_input_tokens`, `max_tokens`, and `capabilities`. Prefer these fields to
hard-coded constants.

Opus 4.6 and Sonnet 4.6 can raise the single-turn output limit to 300,000 with:

```text
Anthropic-Beta: output-300k-2026-03-24
```

## Workload identity and inference location

Workload Identity Federation is GA. Configure OIDC issuers and federation
rules in the Console, then use an SDK to exchange and refresh short-lived
credentials rather than distributing static API keys.

For models released after February 1, 2026, `inference_geo` may request
US-only inference at 1.1 times standard pricing.

## AWS-hosted surfaces

Claude Platform on AWS uses Anthropic-managed infrastructure with AWS billing
and IAM. Through native AWS endpoints, it exposes Messages, Files, Message
Batches, Managed Agents, Agent Skills, code execution, and tool-use APIs.

Amazon Bedrock is a separate AWS-managed surface. Its
`/anthropic/v1/messages` endpoint uses the first-party Messages request shape.
Opus 4.7 and Haiku 4.5 are self-serve there across global and regional
endpoints.

Organizations on Claude Platform on AWS start in the Start tier and do not
advance usage tiers automatically. Billing uses AWS Marketplace; spend limits
are under Billing rather than Limits. The normal increase-request flow is
unavailable, so arrange higher limits through an account representative or
support.

## MCP tunnel migration

MCP tunnel management moved from the Admin API route
`/v1/organizations/tunnels` to `/v1/tunnels` on the Claude API. The new route
requires both:

```text
Anthropic-Beta: mcp-tunnels-2026-06-22
WIF scope: workspace:manage_tunnels
```

The old route remains only during a migration window.

## Continuous rate limits

The Messages API independently enforces requests per minute, input tokens per
minute, and output tokens per minute with continuously replenished token
buckets. Enforcement may occur at sub-minute intervals.

A sharp traffic increase may trigger an acceleration-limit HTTP 429 even when
the steady-state rate looks valid. Ramp traffic gradually and obey
`retry-after`.

## Token accounting

For most models, input-token rate usage is:

```text
ITPM = input_tokens + cache_creation_input_tokens
```

`input_tokens` counts only content after the final cache breakpoint.
Total input is:

```text
cache_read_input_tokens + cache_creation_input_tokens + input_tokens
```

Cache reads normally do not count against ITPM. Claude Haiku 3.5 is the
exception and includes them.

ITPM is estimated when a request starts and reconciled to actual input during
processing. OTPM is charged in real time for generated tokens; the requested
`max_tokens` does not reserve output capacity.

## Shared and dedicated pools

- Opus 4.5 through 4.8 share one combined Opus 4.x pool.
- Sonnet 4.5 and 4.6 share one Sonnet 4.x pool.
- Opus 5 and Sonnet 5 each have separate pools.
- `inference_geo: "us"` and `"global"` use the same capacity.
- Supported `speed: "fast"` calls use dedicated limits rather than the
  standard Opus pool.

A fast-pool throttle returns HTTP 429 with `retry-after`. Its state appears in
`anthropic-fast-*` response headers.

## Batch and agent pools

Message Batches have a model-independent pool:

- 1,000 API requests per minute.
- Up to 200,000 constituent batch requests awaiting successful processing.
- Up to 100,000 constituent requests in a single batch.

Each constituent item, not only the enclosing batch, consumes queue capacity.

Managed Agents have a separate organization-level pool: create operations are
limited to 300 requests per minute, while retrieve, list, stream, and other
read operations are limited to 1,200 requests per minute.

## Spend caps

Calendar-month standard caps are:

| Tier | Cap |
| --- | ---: |
| Start | $500 |
| Build | $1,000 |
| Scale | $200,000 |

Reaching a cap pauses API use until the next month unless it is raised.
Custom-tier organizations have no standard monthly cap. Any organization may
configure a lower self-imposed cap.

## Workspace safeguards

A non-default workspace may set lower RPM, ITPM, OTPM, and spend ceilings.
An unset workspace limit inherits the organization limit. Unused workspace
capacity remains available to other workspaces.

The default workspace cannot be capped. The organization ceiling still
applies even if configured workspace limits sum to more than that ceiling.

## Response headers

`retry-after` is the number of seconds before a retry can succeed.

The API also emits these rate-limit header families:

```text
anthropic-ratelimit-requests-{limit|remaining|reset}
anthropic-ratelimit-tokens-{limit|remaining|reset}
anthropic-ratelimit-input-tokens-{limit|remaining|reset}
anthropic-ratelimit-output-tokens-{limit|remaining|reset}
```

Reset values are RFC 3339 timestamps for full bucket replenishment. Remaining
token values are rounded to the nearest thousand.
