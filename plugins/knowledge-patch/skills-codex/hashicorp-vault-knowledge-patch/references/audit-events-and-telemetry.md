# Audit, events, reporting, and telemetry

Use this reference when changing audit devices or parsers, consuming event streams, querying client activity, producing utilization or billing reports, or alerting on server metrics.

## Audit request headers

- Audit records include the incoming `User-Agent` without HMAC by default.
- Configure whether it is HMACed at `/sys/config/auditing/request-headers/user-agent`.
- Treat a change as an audit-schema and disclosure change; update both parser expectations and retention review.

## File audit-device permissions

- An executable file audit device became an unseal blocker in 1.19.7.
- From 1.19.16, unseal warns about executable bits on an existing device and clears them.
- Creation of a new file audit device still rejects executable permissions. Provision the file with non-executable mode before enabling the device.

## Supplemental audit data

Response audit structures can carry `supplemental_audit_data` for details from non-JSON protocols (since 2.0). `audit-non-hmac-request-keys` and `audit-non-hmac-response-keys` control HMAC treatment of these values. PKI OCSP data uses this field and remains HMACed by default.

## Event subscription routing

- Enterprise event subscriptions on performance standbys no longer require redirection to the active node.
- Events are forwarded only when a matching subscriber exists.
- Subscription authorization checks the event metadata `path` rather than requiring `data_path`.
- Enterprise can forward lease-event notifications from primary to secondary clusters (since 2.0).
- Secret deletion subscriptions do not require a root token.

## Consistent reads after events

An event with metadata `modified=true` also carries `vault_index` (since 1.20). Pass that index through the appropriate client-consistency mechanism when a consumer immediately reads storage affected by the event.

## Event reliability and rotation evidence

- Enterprise 1.19.x can miss events when multiple clients are connected; apply the release-note workaround and monitor gaps.
- LDAP secrets emits rotation-success and rotation-failure events (since 1.21).
- Server logs include details for successful and failed root, database static-role, and LDAP static-role rotation, providing an attestation trail.

## Activity query behavior

- `sys/internal/counters/activity` includes `mount_type`.
- Supplied start and end timestamps are aligned to billing periods; the end is capped at the last completed month (since 1.20).
- Enterprise current-month queries return actual new-client values.
- The activity export schema renames `timestamp` to `token_creation_time` and adds the time of a client's first use within the requested period (since 1.21). Update field mappings rather than reading both names indefinitely.
- Enterprise cumulative namespace counts are available at `sys/internal/counters/activity/cumulative`.
- `/sys/internal/counters/tokens` is deprecated and returns `403 unsupported path`; move integrations to supported activity APIs.

## Client telemetry and dashboards

- Enterprise metric `vault.client.billing_period.activity` reports a cluster-wide distinct-client count and updates every ten minutes (since 1.20).
- The Enterprise client-count dashboard includes a **Client list** view for inspecting the clients behind an aggregate (since 1.21).

## Utilization reports

- Enterprise `/sys/utilization-report` returns a high-level utilization snapshot (since 1.20).
- HCL `development_cluster` defaults to false and is included in utilization output.
- Reports accept namespace filters and include finer Secret Sync detail (since 1.21).
- Manual bundles from `vault operator utilization` rename `snapshots` to `snapshot_records` (since 2.0).
- Each record's `decoded_snapshot` contains the former readable snapshot content.
- Reports also expose `issuer`, `edition`, `add_ons`, `license_start_time`, `license_expiration_time`, and `license_termination_time`.

## Billing APIs

- Enterprise issued-certificate counts are available at `sys/billing/certificates` (since 1.21).
- `sys/billing/overview` returns current- and previous-month metrics, accepts `start_month` and `end_month`, and is available in the admin namespace (since 2.0).
- Internal billing history defaults to 37 months.
- `sys/billing/config` accepts retention from 13 months through six years.

## License utilization fields

The billing overview dashboard and API add the following usage prefixes, each with `.current_month_estimate` and `.previous_month_complete` forms:

- `gcp_kms_operation_count`
- `normalized_oidc_tokens_issued`
- `normalized_spiffe_jwt_token_units`
- `os_local_static_max_role_count`
- `normalized_external_ca_cert_units`

Treat these as explicit schema fields when building exports; do not derive names from GUI labels.

## Product usage metrics

- Product reporting can include anonymous numeric feature-usage data in utilization reports.
- Product metrics include `secret.engine.os.local.account.static.role.count` (since 2.0).
- Metrics also count agent registrations and OAuth resource-server configurations, with separate variants for configurations where RAR is enabled or disabled.

## Response metrics

`vault.core.response_status_code` counts handled status codes and carries `code` and `type` labels (since 1.20). Control label cardinality when relabeling or aggregating it.

## State-report scan

The `sudo`-protected `sys/reporting/scan` endpoint writes Vault-state report files into `reporting_scan_directory` (since 1.21). Ensure the directory is controlled, has sufficient capacity, and is covered by the handling policy for diagnostic data.

## Parser migration checklist

1. Add fixtures for `supplemental_audit_data`, non-HMAC `User-Agent`, and named utilization fields.
2. Replace activity `timestamp` with `token_creation_time` and accept first-use time.
3. Replace `snapshots` with `snapshot_records` and read `decoded_snapshot`.
4. Remove calls to the unsupported token-counter path.
5. Preserve `vault_index` through event-triggered reads.
6. Monitor event gaps when multiple clients or cluster forwarding are involved.
7. Align query windows to billing periods and enforce supported retention bounds.
