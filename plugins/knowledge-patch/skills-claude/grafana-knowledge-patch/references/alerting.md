# Alerting

Use this reference for Grafana-managed and data-source-managed rules,
evaluation, state persistence, recording rules, notification routing,
Alertmanager integration, templates, imports, and alerting APIs.

## Authoring, identity, and provisioning

- In 11.5.0, the alert-rule editor can export a newly created rule as HCL for
  provisioning workflows. Alert-list panels can optionally include inactive
  alerts.
- In 11.6.0, new rules added to groups may receive a UID. Version history can
  restore an earlier rule version and retains the latest version after
  deletion. Rule creation and rule views surface dependency-graph errors, and
  threshold rules support multiple operators.
- In 12.0.0, titles cease to be unique and rules within a group may evaluate
  sequentially. Identify rules by UID. Deleted rules can be recovered, while
  permanently deleted rules are removed separately.
- The 12.0.0 API can convert submitted Prometheus rules to Grafana-managed
  rules; imports of data-source-managed rules skip rules managed by plugins.
- In 12.1.0, Grafana-managed alerting imports Prometheus YAML. Imports preserve
  group labels and `query_offset`, and imported rules evaluate sequentially.
  The Prometheus Rules API gains health and contact-point filters and exposes
  provenance.
- In 12.2.0, the Prometheus-to-Grafana conversion API is stable, accepts extra
  labels, and can return JSON. File provisioning accepts `keepFiringFor` and
  `missing_series_evals_to_resolve`; it refuses to delete a `TimeInterval`
  still referenced by `ActiveTimings`.
- In 12.3.0, otherwise identical rules can be matched by position.
  Provisioning accepts recording rules without conditions but rejects
  receivers with empty names.
- In 12.4.0, Prometheus-rule imports can use a configured default data source;
  only administrators can use the import UI. Rules cannot be saved into
  Git-synced folders.
- In 13.0.0, legacy alert-rule provisioning endpoints are deprecated.
  Provisioning accepts resource-specific permissions, enforces protected-field
  authorization, and notification APIs enforce provenance permissions.
- In 13.1.0, legacy storage accepts label selectors in `AlertRule` and
  `RecordingRule`, and Grafana-managed rules may be created without a group.
  `alerting.rulesAPIV2` enables Rules API v2, which the panel rule drawer uses.

## Evaluation, recovery, and state

- In 11.5.0, `state_periodic_save_batch_size` controls periodic alert-state
  save batches. The `max_attempts` default becomes 3, so upgrades must account
  for the changed retry behavior.
- In 11.6.0, `alertingSaveStateCompressed` enters public preview and
  `alertingNoNormalState` is removed. A missing group requested through
  `/api/ruler/grafana/api/v1/rules/{Namespace}/{Groupname}` returns HTTP 404.
- In 12.0.0, `MissingSeriesEvalsToResolve` /
  `missing_series_evals_to_resolve` controls the missing-series resolution
  threshold. `keep_firing_for` and the `Recovering` state keep rules firing
  during a recovery window.
- In 12.0.0, a single-data-source alert's `$value` becomes the query value.
  Review notification templates that relied on its older representation.
  Simple-condition alert thresholds accept negative values, and expressions
  work for backend-only plugins with `backend: true` and `alerting: false`.
- In 12.1.0, alert instances persist `FiredAt`; the state-history backend can
  write the `ALERTS` metric. Grafana resends states absent from an evaluation
  result and immediately notifies for Error-to-Normal and NoData-to-Normal
  transitions.
- In 12.2.0, evaluation retries with backoff.
  `alertingSaveStateCompressed` is enabled by default, making compressed state
  persistence the default.
- In 12.3.0, periodic state writes support jitter to avoid synchronized
  database load. Starting with 12.3.3, expanded notification-template output
  has a size limit; review unusually large templates.
- In 12.4.0, the pending period also applies to NoData and Error alerts. Alert
  labels become annotation tags, and webhook `valueString` includes expression
  type information.
- In 13.0.0, Alerting adds single-node evaluation. Alert Activity adds saved
  searches, policy filtering, and notification and silence views. Panel-based
  rule creation is behind `createAlertRuleFromPanel`, and Prometheus Rules API
  results can sort by full folder path.
- In 13.1.0, plugin rule origin is propagated in `X-Rule-Origin`.

## Recording rules and remote writes

- In 11.6.0, each recording rule can select its write target instead of relying
  only on a shared destination.
- In 12.1.0, recording rules become enabled by default. Grafana-managed
  recording rules support PDC and use `default_datasource_uid` as the default
  target; the UI can disable writes per data source.
- In 12.2.0, Grafana strips private labels before writing recording rules.
- In 12.4.0, alerting remote writes use data-source headers.

## Routing, contact points, and delivery

- In 11.6.0, Jira becomes an alerting integration, including for cloud
  Alertmanagers, and Slack receivers gain a color option. Dashboard and panel
  URLs generated by alert templates include the alert time range.
- In 12.0.0, webhook receivers can sign notifications with HMAC and render
  templatable payloads.
- In 12.1.0, webhook receivers support OAuth2. Grafana can send SMTP
  configuration to a remote Alertmanager.
- In 12.4.0, the default notification configuration uses an empty receiver.
  Single-Alertmanager contact points gain versioning, and receiver tests move
  to Kubernetes-style App Platform APIs.
- In 12.4.0, OpsGenie is deprecated. External Alertmanager connections support
  client-certificate authentication and other TLS options.
- In 13.0.0, managed route assignment moves from labels to
  `notification_settings.policy`; managed routes gain access control. Legacy
  notification actions are deprecated.
- In 13.1.0, administrators can restrict available contact-point integration
  types and limit email recipients to organization members. Notification
  provisioning endpoints are deprecated.

## Alertmanager configuration, HA, and synchronization

- In 11.5.0, `alertingApiServer` is enabled by default. Notification-policy
  trees and previews use the Kubernetes API; template-group API and UI results
  include built-in default templates.
- In 12.0.0, manual edits and restores of internal Alertmanager configuration
  are no longer allowed through settings, and its internal POST endpoint is
  removed. Alertmanager requests can supply `reqAction` for RBAC checks.
- In 12.1.0, alerting HA supports Redis Sentinel.
- In 12.1.0, cloud migration treats mute timings as notification-policy
  dependencies, so the resources migrate together.
- In 12.4.0, the Grafana HA Alertmanager cluster-metrics prefix changes. Update
  dashboards and selectors. `grafana_alerting_rule_group_rules` gains a
  `folder_uid` label.
- During the 13.0-upgrade, these endpoints are removed:
  `DELETE /api/alertmanager/grafana/config/api/v1/alerts` and
  `POST /api/alertmanager/grafana/config/api/v1/receivers/test`.
  `GET /api/alertmanager/grafana/config/api/v1/alerts`,
  `GET /api/alertmanager/grafana/config/history`, and
  `POST /api/alertmanager/grafana/config/history/{id}/_activate` become
  admin-only.
- Replace that automation with
  `/apis/notifications.alerting.grafana.app/v1beta1/namespaces/{namespace}/`
  resources: `receivers`, `routingtrees`, `templategroups`, `timeintervals`,
  and `inhibitionrules`.
- During the 13.0-upgrade,
  `GET /api/alertmanager/grafana/api/v2/status` begins requiring
  `alert.notifications.system-status:read`, not
  `alert.notifications:read`. Add it to custom roles; administrators obtain it
  through `fixed:alerting.notifications:writer`.
- In 13.1.0, the settings UI exposes Mimir Alertmanager auto-synchronization.
  External Alertmanager sender metrics identify data sources by UID.

## Removed alerting gates

- In 12.0.0, `alertingNoDataErrorExecution`, the Loki Alert State History
  toggles, and `traceQLStreaming` are removed along with other unrelated
  toggles. Stop using them to gate behavior.
- In 13.0.0, `kubernetesAlertingRules` is removed.
- In 13.1.0, `alertRuleUseFiredAtForStartsAt` is removed.

## Enterprise alerting

- In 12.3.0, per-rule enrichment endpoints and rule-view components are
  available. An enrichment mutator can attach rule-UID labels for efficient
  label-selector matching.
- In 12.4.0, enrichment read and write actions become separate RBAC
  permissions, and template testing receives a dedicated permission.
