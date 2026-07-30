# Audit, Events, Billing, and UI

## Audit records and devices

- Audit records include incoming `User-Agent` without HMAC by default.
  Configure HMAC behavior at
  `/sys/config/auditing/request-headers/user-agent`.
- Executable file audit devices became an unseal blocker in 1.19.7. From
  1.19.16, unseal warns and clears existing executable bits; creation of a new
  executable file audit device remains forbidden.
- Response audit entries can carry `supplemental_audit_data` in request and
  response structures for non-JSON protocols.
- `audit-non-hmac-request-keys` and `audit-non-hmac-response-keys` govern
  supplemental values. PKI OCSP details use this field and remain HMACed by
  default.
- `enable_metadata_on_failures` can expose client-certificate metadata in
  failed-login responses and audit entries.

## Event subscriptions

- Enterprise event subscriptions on performance standbys no longer redirect to
  the active node. Events are forwarded only when a matching subscriber
  exists.
- Authorization uses event metadata `path`, not a required `data_path`.
- Secret-deletion subscriptions no longer require a root token.
- Storage-changing events with metadata `modified=true` include `vault_index`
  so consumers can apply client-consistency controls to follow-up reads.
- LDAP secrets emits events such as rotation success and failure.
- Enterprise adds lease events and can forward notifications from primary to
  secondary clusters.
- Multiple event clients can miss events in Enterprise 1.19; use the documented
  workaround.

## Activity and client counting

- `sys/internal/counters/activity` includes `mount_type`.
- Supplied activity start/end times align to billing periods, and the end is
  capped at the last completed month. Enterprise current-month queries return
  actual new-client values.
- Activity export renames `timestamp` to `token_creation_time` and adds the
  client's first-use timestamp for the requested period.
- Enterprise adds cumulative namespace counts at
  `sys/internal/counters/activity/cumulative` and issued-certificate counts at
  `sys/billing/certificates`.
- Enterprise `vault.client.billing_period.activity` is a cluster-wide distinct
  client count updated every ten minutes.
- The client-count dashboard has a **Client list** tab for inspecting clients
  represented in an aggregate.

## Utilization and billing

- Enterprise `/sys/utilization-report` returns a high-level snapshot.
  HCL `development_cluster` defaults to false and is included in reports.
- Reports add namespace filtering and finer Secret Sync detail.
- Product usage reporting contributes anonymous numeric feature-use data to
  utilization reports.
- `sys/billing/overview` returns current- and previous-month metrics, accepts
  `start_month` and `end_month`, and is available in the admin namespace.
- The same billing-overview data is available through a GUI dashboard.
- Internal billing history defaults to 37 months. `sys/billing/config` accepts
  retention from 13 months through six years.
- Manual `vault operator utilization` bundles replace `snapshots` with
  `snapshot_records`; `decoded_snapshot` contains the previous readable
  snapshot.
- Reports add `issuer`, `edition`, `add_ons`, `license_start_time`,
  `license_expiration_time`, and `license_termination_time`.

## Metrics and telemetry

- `vault.core.response_status_code` counts handled status codes with `code` and
  `type` labels.
- License utilization fields include:
  `gcp_kms_operation_count`, `normalized_oidc_tokens_issued`,
  `normalized_spiffe_jwt_token_units`, `os_local_static_max_role_count`, and
  `normalized_external_ca_cert_units`. Each has
  `.current_month_estimate` and `.previous_month_complete`.
- Product metrics add `secret.engine.os.local.account.static.role.count`,
  agent-registration counts, and OAuth resource-server configuration counts.
  Agent and resource-server metrics have variants for RAR enabled and disabled.
- Successful and failed automated root, database static-role, and LDAP
  static-role rotations emit detailed server logs suitable for attestation.

## Reporting scan

The sudo-protected `sys/reporting/scan` endpoint writes state-report files to
the configured `reporting_scan_directory`.

## GUI behavior

- Enterprise can configure primary and backup login-form methods.
  `/vault/auth?with=` denotes only an auth mount path and renders a simplified
  form; choosing another method no longer rewrites the parameter.
- Enterprise namespace picker search and navigation do not require
  reauthentication. Namespace onboarding can begin from a guided
  questionnaire.
- Community Edition GUI can list and add TOTP accounts, reveal hidden codes,
  and display expiry timers.
- Enterprise GUI configures workload identity federation for AWS, Azure, and
  GCP.
- Secrets-engine paths move from `/secrets` to `/secrets-engines`; list-view
  bulk deletion is removed. TLS-certificate login is available.
- The visual policy editor generates ACL policy snippets.
- In 1.21 and 2.0, changing **Items per page** from a later Secrets Engines
  page can show an empty or partial table. Return to page 1 or refresh there.
- An Enterprise 2.0 EGP can block a root token's child-namespace GUI request to
  `sys/internal/ui/mounts`; use CLI/API or explicitly allow the endpoint.
