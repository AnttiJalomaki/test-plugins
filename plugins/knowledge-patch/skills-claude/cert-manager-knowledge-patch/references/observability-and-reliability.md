# Observability and Reliability

Use this reference for metrics and logs, CA bundle rotation, webhook recovery,
and failure modes that changed from loops or terminal errors to bounded retry.

## Logging

### Structured context affects literal matches

Since `upgrade-1.17`, log messages include more contextual structured data.
Rules that match complete log lines or literal message strings may stop
matching. Parse structured fields or use stable substrings and conditions
instead of treating the rendered line as an API.

## Certificate and challenge metrics

### Validity timestamps

Since `1.18`, these metrics expose certificate issuance and expiration times:

- `certmanager_certificate_not_before_timestamp_seconds`
- `certmanager_certificate_not_after_timestamp_seconds`

Use them for remaining-lifetime alerts while accounting for renewal policy and
clock skew.

### Challenge status

Since `1.19`, `certmanager_certificate_challenge_status` exposes ACME challenge
status for dashboards and alerts.

## ACME request metric migration

In `upgrade-1.19`, the `path` label was removed from
`certmanager_acme_client_request_count` and
`certmanager_acme_client_request_duration_seconds`. It was replaced by the
bounded-cardinality `action` label. Rewrite PromQL queries, dashboards, and
alerts that select or group by `path`. If preserving the old high-cardinality
semantics is essential, implement it explicitly with Prometheus relabeling or a
recording rule.

## Helm monitoring labels and endpoints

### Stable metrics label

In `1.20`, enabling Prometheus monitoring always sets the metrics label to
`cert-manager`; it no longer varies with the installation namespace or Helm
release name. Update selectors that interpolated either value.

### Removed monitor overrides

In `upgrade-1.21`, these chart values were removed:

- `prometheus.servicemonitor.targetPort`
- `prometheus.servicemonitor.path`
- `prometheus.podmonitor.path`

Delete them before upgrade or chart schema validation fails. Metrics now use
the fixed `/metrics` path and `http-metrics` port name. Custom scrape
configuration that used `tcp-prometheus-servicemonitor` must switch to the new
port name.

## Cainjector CA bundle behavior

### Merge lifecycle

In `1.17`, the opt-in `CAInjectorMerging` gate made cainjector add new CA
certificates to an injected bundle rather than replacing the existing
certificate. This supplies a trust overlap during issuer rotation.

In `1.19`, the gate became beta and enabled by default. It could still be
explicitly disabled when replacement semantics were required.

In `1.21`, merging became GA and unconditional. The feature gate can no longer
restore replacement behavior. Cainjector also always uses server-side apply,
and the `ServerSideApply` feature gate is deprecated.

### Ignore selected namespaces

In `1.21`, pass `--ignore-namespaces` to cainjector to exclude specified
namespaces when it watches Secrets for CA injection.

## Webhook recovery

In `1.21`, the webhook uses wall-clock polling to detect a serving-certificate
renewal missed during host suspension or VM live migration. It recovers within
one minute after resume.

## Bounded failure behavior

### Public-key mismatch

Starting in 1.18.5, an issuer response whose certificate public key does not
match its CSR is rejected before storage. Issuance backs off instead of
entering an infinite reissuance loop.

### Already-expired certificate

In `1.21`, a certificate response that is already expired no longer produces
an infinite reissuance loop.

### Transient ACME operations

In `1.21`, TLS handshake timeouts, DNS failures, and context cancellation
during nonce fetches or authorization waits retry through workqueue backoff
rather than terminally failing the Challenge.

### DigitalOcean diagnostics

In `1.20`, DigitalOcean DNS-01 retries are regulated and complete provider
errors are attached to the Challenge as events. Inspect events before reducing
backoff or retry settings.
