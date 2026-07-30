# Billing, quotas, and traffic pools

Source batch: `limits-and-billing`.

Use this reference for account setup, capacity planning, spend controls, retry
behavior, and Batch or Priority traffic sizing.

## Billing-account tiers and project quotas

The Cloud Billing account determines the billing plan, usage tier, and account
spend cap inherited by every linked project. API keys have no independent
billing settings. Request quotas are enforced per project; all keys in a project
share its counters.

Tier qualification and account caps are:

| Tier | Qualification | Monthly account cap |
| --- | --- | --- |
| Tier 1 | — | $250 |
| Tier 2 | $100 cumulative paid Cloud usage and 3 days since first successful payment | $2,000 |
| Tier 3 | $1,000 cumulative paid Cloud usage and 30 days | $20,000–$100,000 |

Qualifying spend includes all Google Cloud services on the billing account and
upgrades are automatic. Reaching the account cap pauses the API for every linked
project until the billing cycle restarts on the first of the month. Moving a
project to another billing account changes its inherited tier and limits;
unlinking billing returns it to the Free Tier.

## Account-wide Prepay and Postpay

Prepay/Postpay began taking effect March 23, 2026. New users generally default
to Prepay, though rollout accounts may temporarily receive Postpay or a choice.

Prepay purchases range from $10 to $5,000. They apply only to API usage, expire
after 12 months, and are non-refundable except when the account moves to
Postpay. A zero balance stops every key in every linked project without
downgrading the projects to Free Tier.

The roughly ten-minute billing pipeline can allow overages or a negative
balance, especially for long-running batches and agents. Auto-reload can prevent
a stop and has a monthly automatic-charge ceiling; manual purchases do not count
toward that ceiling.

Eligible promotional Cloud credits are consumed before prepaid funds, but only
while the Prepay balance remains positive. The $300 Welcome credit issued after
March 2, 2026 is ineligible. A delinquent charge for another Cloud service can
suspend API access even when prepaid funds remain.

Tier 3 accounts can become eligible to move the entire billing account to
Postpay. The move refunds unused prepaid credits, changes every linked project,
and cannot be reversed for that account. The manual switch control is
temporarily disabled.

## Project spend caps and processing delay

Editors, owners, and admins can set an experimental monthly project cap in AI
Studio. The cap survives moving the project to another billing account, while
the project's accumulated spend resets.

Both a project cap and a zero prepaid balance can be exceeded during the roughly
ten-minute processing delay. Long-running work can continue to accrue charges
during that window; do not treat the cap as an instantaneous kill switch.

## Interactive quota dimensions and spend rate

Interactive limits independently evaluate:

- requests per minute;
- input tokens per minute;
- requests per day, reset at midnight Pacific time.

Exceeding any dimension fails the request. Limits are per project rather than
per key. Preview and experimental models generally have tighter limits. Active
values in AI Studio can change with tier and account status.

Paid tiers may also have a rolling ten-minute spend-rate limit: $10 for Tier 1
and $200 for Tiers 2 and 3. Crossing it returns
`429 RESOURCE_EXHAUSTED` even if RPM and TPM remain available.

Requests that fail with HTTP 400 or 500 are not billed for tokens but still
consume quota. `GetTokens` requests are neither billed nor counted against
inference quota.

## Priority and Batch pools

Priority inference has its own default limit at 0.3 times the corresponding
model-and-tier limit. Its consumption also counts toward overall interactive
traffic.

Batch requests are isolated from non-batch quotas. Batch permits 100 concurrent
requests, a 2 GB input file, and 20 GB of stored files. Enqueued-token limits
apply per model across all active jobs.

Representative Tier 1 / Tier 2 / Tier 3 enqueued-token ceilings are:

| Model family | Tier 1 | Tier 2 | Tier 3 |
| --- | ---: | ---: | ---: |
| Gemini 3.6 Flash and 3.5 Flash | 3M | 400M | 1B |
| Gemini 3.5 Flash-Lite and 3.1 Flash Lite | 10M | 500M | 1B |
| Gemini 3.1 Pro Preview and 2.5 Pro | 5M | 500M | 1B |
| Gemini 2.5 Pro TTS | 25K | 100K | 1M |
| Gemini 2.5 Flash TTS | 100K | 100K | 4M |
| Gemini Embedding | 500K | 5M | 10M |
