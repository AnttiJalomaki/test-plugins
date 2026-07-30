# Alerting

Use this reference for Grafana-managed rules, data-source-managed imports,
evaluation and state persistence, notification delivery, Alertmanager
integration, and alerting authorization.

## Rule identity, authoring, and lifecycle

- The alert-rule editor can export a newly created rule as HCL, and alert-list
  panels can include inactive alerts (since 11.5.0).
- Rule creation and views expose dependency-graph errors, and threshold rules
  support multiple operators (since 11.6.0).
- A new rule inserted into a group may receive a UID. Rule version history can
  restore an earlier version and retains the latest version after deletion
  (since 11.6.0).
- Grafana 12.0.0 removes the title uniqueness constraint and supports
  sequential evaluation inside a group. Identify a rule by UID, not title.
- Deleted rules can be recovered; permanently deleted rules are purged through
  a separate lifecycle path (since 12.0.0).
- Simple-condition thresholds accept negative values (since 12.0.0).
- Otherwise-identical rules can be matched by position, and provisioning
  accepts recording rules without conditions (since 12.3.0).
- Legacy storage accepts label selectors in `AlertRule` and `RecordingRule`,
  and Grafana-managed rules can be created without a group (since 13.1.0).

## Evaluation, state, and recovery

- `state_periodic_save_batch_size` controls periodic state-save batch size,
  and `max_attempts` defaults to `3` (since 11.5.0). Recheck retry load and
  persistence sizing after an upgrade.
- `alertingSaveStateCompressed` entered public preview in 11.6.0 and is enabled
  by default in 12.2.0.
- `MissingSeriesEvalsToResolve` in the API and
  `missing_series_evals_to_resolve` in the form configure how many
  missing-series evaluations resolve an alert (since 12.0.0).
- `keep_firing_for` and the `Recovering` state allow a firing rule to remain
  active during its recovery window (since 12.0.0).
- For a single-data-source alert, `$value` returns the query value (since
  12.0.0). Review templates that expected its older representation.
- Expressions work with plugins declaring `backend: true` and `alerting:
  false` (since 12.0.0).
- Alert instances persist `FiredAt`; the state-history backend can write the
  `ALERTS` metric. Grafana resends states absent from evaluation results and
  immediately notifies for Error-to-Normal and NoData-to-Normal transitions
  (since 12.1.0).
- File provisioning accepts `keepFiringFor` and
  `missing_series_evals_to_resolve`; evaluation retries with backoff (since
  12.2.0).
- Private labels are removed before recording-rule writes (since 12.2.0).
- Periodic state storage can add jitter to spread database writes (since
  12.3.0). From 12.3.3, expanded notification templates have a size limit;
  review unusually large expansions.
- The pending period applies to NoData and Error alerts (since 12.4.0).
  Alert labels become annotation tags, and webhook `valueString` includes
  expression-type information.
- Grafana 13.0.0 adds a single-node evaluation mode. Server-side expression
  pipelines isolate broken nodes and return partial results from unaffected
  nodes.
- Math-expression binary operations gain a memory limit; SQL-expression schema
  queries interpolate variables, and string-to-number conversion preserves
  null and empty strings (since 13.1.0).

## Recording rules and Prometheus imports

- Each recording rule can select its own write destination (since 11.6.0).
- Grafana 12.0.0 exposes an API to convert submitted Prometheus rules to
  Grafana-managed rules. Imports of data-source-managed rules skip rules owned
  by plugins.
- Recording rules are enabled by default in 12.1.0, a breaking default
  change. Grafana-managed recording rules support PDC, use
  `default_datasource_uid` as the default destination, and allow writes to be
  disabled per data source in the UI.
- Grafana-managed alerting can import Prometheus YAML (since 12.1.0). Imports
  evaluate sequentially and preserve group labels and `query_offset`. The
  Prometheus Rules API adds health and contact-point filters and returns
  provenance.
- The Prometheus conversion API is stable in 12.2.0, can return JSON, and
  accepts extra labels.
- Imports can use a configured default data source, while the import UI is
  restricted to administrators (since 12.4.0).

## Notification integrations and payloads

- Jira is available as an alert integration, including in cloud
  Alertmanagers, and Slack receivers accept a color (since 11.6.0).
- Dashboard and panel links generated in alert templates include the alert
  time range (since 11.6.0).
- Webhooks support HMAC signatures and templatable payload bodies (since
  12.0.0).
- Webhook receivers support OAuth2, and Grafana can send SMTP configuration to
  a remote Alertmanager (since 12.1.0).
- Receivers with an empty name are rejected (since 12.3.0).
- In 12.4.0, the default notification configuration uses an empty receiver.
  Contact points in the single-Alertmanager path gain versioning, and receiver
  testing moves to Kubernetes-style App Platform APIs.
- OpsGenie is deprecated (since 12.4.0). External Alertmanager connections
  support client-certificate authentication and TLS settings.
- Grafana 13.1.0 can restrict available contact-point integration types and
  limit email recipients to organization members. The settings UI exposes
  automatic synchronization for Mimir Alertmanager.

## Alertmanager APIs and authorization

`alertingApiServer` is enabled by default from 11.5.0. Notification-policy
trees and previews use the Kubernetes API; template-group API and UI responses
also include the built-in default templates.

A request for a nonexistent group at
`/api/ruler/grafana/api/v1/rules/{Namespace}/{Groupname}` returns HTTP 404
(since 11.6.0). `alertingNoNormalState` was removed in the same release.

Grafana 12.0.0 removes manual editing or restoration of the internal
Alertmanager configuration and removes its internal POST endpoint.
Alertmanager RBAC requests support `reqAction`.

Deleting a `TimeInterval` checks whether `ActiveTimings` still references it
(since 12.2.0). Alert rules cannot be saved in Git-synced folders (since
12.4.0).

During the `13.0-upgrade`:

- `DELETE /api/alertmanager/grafana/config/api/v1/alerts` and
  `POST /api/alertmanager/grafana/config/api/v1/receivers/test` are removed.
- `GET /api/alertmanager/grafana/config/api/v1/alerts`,
  `GET /api/alertmanager/grafana/config/history`, and
  `POST /api/alertmanager/grafana/config/history/{id}/_activate` become
  administrator-only.
- Move automation under
  `/apis/notifications.alerting.grafana.app/v1beta1/namespaces/{namespace}/`.
  Map receivers, policies, templates, mute timings, and inhibition rules to
  `receivers`, `routingtrees`, `templategroups`, `timeintervals`, and
  `inhibitionrules`.
- `GET /api/alertmanager/grafana/api/v2/status` requires
  `alert.notifications.system-status:read`, not
  `alert.notifications:read`. Administrators inherit it through
  `fixed:alerting.notifications:writer`; update custom roles.

## Provisioning permissions and route assignment

Grafana 13.0.0 assigns managed routes through
`notification_settings.policy`, not labels. Managed routes have access
control. Provisioning APIs accept resource-specific permissions and enforce
protected-field authorization; notification APIs enforce provenance
permissions. Legacy notification actions and legacy alert-rule provisioning
endpoints are deprecated.

Alert Activity adds saved searches, policy filtering, and notification and
silence views. Dashboard-panel rule creation is available behind
`createAlertRuleFromPanel`, and the Prometheus Rules API can sort by full
folder path (since 13.0.0).

Grafana 13.1.0 introduces `alerting.rulesAPIV2` and uses Rules API v2 in the
panel alert-rule drawer. Notification provisioning endpoints are deprecated,
so new automation should use the newer API model.

## Enterprise enrichments and permissions

- Enterprise 12.3.0 adds per-rule enrichment endpoints and rule-view
  components. An enrichment mutator can add rule-UID labels for efficient
  label selection.
- Enterprise enrichments gain distinct read and write permissions, and the
  template-testing API gets a dedicated permission (since 12.4.0).
- Alerting remote writes use data-source headers. The
  `grafana_alerting_rule_group_rules` metric adds `folder_uid`, and the Grafana
  HA Alertmanager cluster metric prefix changes (since 12.4.0). Update metric
  selectors and dashboards.
- Plugin rule origin is forwarded in `X-Rule-Origin`, and external
  Alertmanager sender metrics identify data sources by UID (since 13.1.0).

## High availability

Grafana Alerting HA supports Redis Sentinel deployments (since 12.1.0).
Validate failover, state persistence, notification deduplication, and the
renamed HA cluster metrics after upgrades.
