# Limits and Billing

## Distinguish account billing from project quotas

The Cloud Billing account determines the billing plan, usage tier, and account
spend cap inherited by all linked projects (`limits-and-billing`). API keys do
not have independent billing settings. Request quotas are enforced per project,
and all keys in a project share its counters.

Tier eligibility is based on cumulative paid Cloud usage across every Google
Cloud service on the billing account:

| Tier | Paid usage | Time since first successful payment | Monthly account cap |
| --- | ---: | ---: | ---: |
| 1 | Initial paid tier | — | $250 |
| 2 | $100 | 3 days | $2,000 |
| 3 | $1,000 | 30 days | $20,000–$100,000 |

Upgrades are automatic. Reaching the account cap pauses the API for every
linked project until the billing cycle restarts on the first of the month.
Moving a project changes the inherited tier and limits to those of the new
billing account. Unlinking billing returns the project to the Free Tier.

## Handle Prepay and Postpay account-wide

Prepay/Postpay began taking effect March 23, 2026. New users generally default
to Prepay, although rollout accounts can temporarily receive Postpay or a plan
choice.

Prepay purchases:

- range from $10 to $5,000
- apply only to API usage
- expire after 12 months
- are non-refundable except when the account transitions to Postpay

A zero balance stops every API key in every linked project; it does not
downgrade them to the Free Tier. The roughly ten-minute billing pipeline can
allow overage or a negative balance, especially for long-running batches and
agents.

Auto-reload can prevent a stop. Its monthly automatic-charge ceiling excludes
manual purchases.

Eligible promotional Cloud credits are used before prepaid funds, but only
while the Prepay balance remains positive. The $300 Welcome credit issued after
March 2, 2026 is ineligible. A delinquent charge for another Cloud service can
suspend API access even when prepaid funds remain.

Tier 3 billing accounts can become eligible to move the entire account to
Postpay. The transition refunds unused prepaid credits, changes every linked
project, and cannot be reversed for that account. The manual switch control is
temporarily disabled.

## Treat project spend caps as delayed controls

AI Studio editors, owners, and admins can set an experimental monthly cap for a
project. The cap survives moving the project to another billing account, while
accumulated spend resets.

Neither a project cap nor zero prepaid balance is instantaneous. Processing can
lag by roughly ten minutes, and long-running work can accrue charges during the
window. Add application-level budget checks and cancellation where a hard stop
matters.

## Diagnose quota and spend-rate failures

Interactive enforcement independently evaluates:

- requests per minute
- input tokens per minute
- requests per day

Crossing any dimension fails the request. Daily counters reset at midnight
Pacific time. Limits are per project, not per API key. Preview and experimental
models generally have tighter limits, and active values in AI Studio can change
with tier and account status.

Paid tiers can also have a rolling ten-minute spend-rate limit:

| Tier | Rolling ten-minute spend |
| --- | ---: |
| 1 | $10 |
| 2 | $200 |
| 3 | $200 |

Crossing the spend-rate limit returns `429 RESOURCE_EXHAUSTED` even when RPM and
TPM remain.

Requests failing with HTTP 400 or 500 are not billed for tokens, but they still
consume quota. `GetTokens` is neither billed nor counted against inference
quota.

## Size priority and batch pools separately

Priority inference has a separate default limit equal to 0.3 times the
corresponding model-and-tier limit. Its usage also counts toward overall
interactive traffic.

Batch requests are isolated from non-batch quotas. Batch limits include:

- 100 concurrent requests
- 2 GB maximum input file
- 20 GB stored files
- an enqueued-token limit applied per model across all active jobs

Representative Tier 1 / Tier 2 / Tier 3 enqueued-token ceilings are:

| Model family | Tier 1 | Tier 2 | Tier 3 |
| --- | ---: | ---: | ---: |
| Gemini 3.6 Flash and 3.5 Flash | 3M | 400M | 1B |
| Gemini 3.5 Flash-Lite and 3.1 Flash Lite | 10M | 500M | 1B |
| Gemini 3.1 Pro Preview and 2.5 Pro | 5M | 500M | 1B |
| Gemini 2.5 Pro TTS | 25K | 100K | 1M |
| Gemini 2.5 Flash TTS | 100K | 100K | 4M |
| Gemini Embedding | 500K | 5M | 10M |
