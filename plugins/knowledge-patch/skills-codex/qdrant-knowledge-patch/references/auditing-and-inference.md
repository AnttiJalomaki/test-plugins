# Auditing and Inference

## Record Protected API Operations

Qdrant can audit API operations that require authentication or authorization
(since 1.17.0). Enable auditing when protected actions need an operational
record.

Audit coverage is tied to authenticated or authorized API activity; do not
treat it as a record of every unauthenticated process event.

## Query Audit Records Across the Cluster

Use the audit-log query endpoint to aggregate entries from every cluster node
(since 1.18.0). Returned details include:

- timestamp;
- API method;
- authentication type;
- access result;
- client information.

Filter results by time range or by any recorded field value. This avoids
inspecting log files independently on each server.

## Correlate Audits with Request Traces

Audit entries record a caller-supplied tracing identifier when the request
contains any of these headers (since 1.18.0):

- `x-request-id`;
- `x-tracing-id`;
- `traceparent`.

Propagate one of these values from the application to correlate the audited
operation with client-side and distributed-tracing logs.

## Supply Inference Credentials per Request

External inference-provider API keys can be passed in request headers (since
1.17.0). Use request-scoped headers when credentials must accompany individual
inference requests instead of being shared as one server-side value.
