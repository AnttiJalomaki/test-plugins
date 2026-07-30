# Service Tiers

Use this reference to configure Flex and Priority, distinguish project defaults from per-request choices, and build retry and traffic-ramp behavior around effective processing tiers.

## Flex economics and endpoint support

Flex is available on Responses and Chat Completions. It uses Batch API token rates and retains prompt-cache discounts.

Official SDK requests default to a ten-minute timeout and automatically retry `408 Request Timeout` twice. Long-running Flex work can outlive that client timeout, so raise the timeout globally or per request when the workload requires it.

```python
response = client.with_options(timeout=900.0).responses.create(
    model="<supported-model>",
    input="<long-running task>",
    service_tier="flex",
)
```

Choose the timeout from expected work duration rather than relying on the default retry loop to extend the same request indefinitely.

## Flex capacity failures

When Flex lacks capacity, the API returns `429 Resource Unavailable` and does not charge for that request.

Two retry strategies have different cost behavior:

- Retry Flex with exponential backoff to retain Flex pricing.
- Retry with `service_tier: "auto"`, or omit the field, to use the project's default processing mode.

Do not treat this capacity response as proof that the request payload or normal rate-limit configuration is invalid.

## Selecting Priority

Priority can be requested explicitly:

```json
{
  "model": "<supported-model>",
  "input": "Latency-sensitive request",
  "service_tier": "priority"
}
```

A project can also make Priority the default for requests that omit `service_tier`. The project-level transition happens gradually, so omission does not imply that every request immediately uses Priority.

Inspect the response's `service_tier` field to determine the tier that actually processed each request. Use this effective field for latency and billing telemetry rather than inferring it from request configuration.

## Rate and ramp behavior

Standard and Priority traffic share the same per-model rate limit.

At traffic of at least one million tokens per minute, increasing TPM by more than 50 percent within 15 minutes can trigger the Priority ramp limit. Affected Priority requests are processed with `service_tier: "default"` and billed at Standard rates.

Ramp sustained traffic gradually. Also monitor the effective response tier so fallback is visible rather than mistaken for unexplained Priority latency drift.

## Priority compatibility

Priority:

- Retains prompt-cache discounts.
- Supports multimodal image inputs.
- Does not support long-context requests.
- Does not support fine-tuned models.
- Does not support embeddings.
- Adds a per-token premium.

Priority is intended for steady, latency-sensitive traffic. Flex or other modes are better aligned with erratic batch and evaluation workloads where per-token price matters more than steady latency.

## Operational checklist

1. Set longer timeouts for genuinely long Flex operations.
2. Handle `429 Resource Unavailable` separately from payload errors and choose whether to retain Flex pricing.
3. Record requested and effective `service_tier` values.
4. Account for a gradual project-default transition.
5. Keep sustained Priority traffic ramps within the stated threshold.
6. Reject or reroute unsupported Priority workloads before sending them.
7. Include cache discounts, Priority premium, and fallback billing in cost telemetry.
