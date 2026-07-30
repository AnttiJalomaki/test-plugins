# Observability And Service Integrations

CloudWatch, Logs, Synthetics, AppConfig, AppSync, Backup, EMR, and other service integrations. Items are organized by topic; parenthetical identifiers preserve exact extraction-batch attribution.

## Amplify build compute types (`2025-10`)

Amplify constructs support configuring the build compute type.

## AppConfig configuration-profile deletion protection (`2025-05`)

AppConfig configuration profiles now participate in deletion-protection checks.

## AppConfig deployment-tick extensions (`2025-01`)

AppConfig L2 extensions support the `atDeploymentTick` action point.

## AppConfig environment deletion protection (`2025-01`)

AppConfig environments can opt into deletion protection.

## AppSync data-source integrations (`2025-04`)

AppSync constructs support data-source integrations.

## AppSync enhanced metrics (`2026-03`)

AppSync GraphQL APIs support `EnhancedMetricsConfigProperty`.

## AppSync Events (`2025-02`)

AppSync Events has L2 constructs.

## Backup schedule time zones (`2025-07`)

Backup constructs support `ScheduleExpressionTimezone`.

## CloudWatch anomaly-detection alarms (`2025-05`)

CloudWatch constructs support anomaly-detection alarms.

## CloudWatch Logs deletion protection (`2026-01`)

CloudWatch Logs constructs support deletion-protection configuration.

## CloudWatch Logs field indexes (`2025-03`)

The Log Group L2 construct accepts `fieldIndexPolicies`.

## CloudWatch Logs transformers (`2025-07`)

CloudWatch Logs constructs support log transformers.

## CloudWatch metric identity and visibility (`2025-07`)

CloudWatch `Metric` exposes `id` and `visible` properties.

## CloudWatch pie-chart labels (`2025-09`)

CloudWatch pie charts can display labels.

## CloudWatch quoted metric-math strings (`2026-06`)

CloudWatch metric-math validation no longer treats quoted strings as unknown identifiers.

## Cross-account CloudWatch widgets (`2025-07`)

CloudWatch log-query and metric widgets accept an account ID for cross-account visibility.

## Dashboard query languages (`2025-07`)

CloudWatch dashboard constructs expose the `queryLanguage` property.

## EMR instance-fleet priority allocation (`2026-05`)

EMR instance fleets support priority allocation.

## Expanded EMR create-cluster options (`2025-09`)

`EmrCreateClusterOptions` accepts `ebsRootVolumeIops`, `ebsRootVolumeThroughput`, and `managedScalingPolicy`.

## Infrequent Access logs in ADC regions (`2025-07`)

The CloudWatch Logs Infrequent Access log class is supported in ADC regions.

## MediaConnect L2 constructs (`2026-07`)

MediaConnect now has L2 construct support.

## MediaPackage v2 region and name handling (`2026-04`)

MediaPackage v2 resources expose a region attribute and apply additional naming validation.

## Metric filters on transformed logs (`2025-10`)

CloudWatch Logs metric filters can opt to run against transformed logs.

## Metric-filter dimension maps (`2025-06`)

Metrics exposed from CloudWatch Logs metric filters now retain the filter's dimension map.

## Metric-filter dimensions reverted (`2025-07`)

Metrics exposed from CloudWatch Logs metric filters no longer retain the filter's dimension map; this reverts the behavior added in the previous batch.

## Multiple stats commands in log queries (`2025-07`)

CloudWatch Logs query strings can contain multiple `stats` commands.

## New Synthetics Python runtimes (`2025-05`)

The Synthetics runtime catalog includes Python canary runtimes 5.0 and 5.1.

## New Synthetics runtimes (`2025-01`)

The Synthetics runtime catalog now includes Node Playwright 1.0 and Python Selenium 4.1.

## New Synthetics runtimes (`2026-01`)

The Synthetics runtime catalog adds `syn-nodejs-3.0`, Playwright 4.0 and 5.0, and Puppeteer 12.0 and 13.0.

## New Synthetics runtimes (`2026-05`)

The Synthetics runtime catalog includes Playwright 5.1 and 6.0.

## PromQL alarms (`2026-05`)

CloudWatch provides a PromQL Alarm L2 construct.

## Regex JSON metric filters (`2025-02`)

CloudWatch Logs JSON metric filters support regular-expression patterns.

## Region Info additions (`2025-07`)

Region Info supports the `eusc-de` and `ap-southeast-6` regions.

## Root-level Synthetics scripts (`2025-10`)

Synthetics canary assets may place script files at the archive root when using Puppeteer runtime version 11 or later.

## Safe canary updates (`2025-06`)

Synthetics constructs support safe canary updates.

## Search expressions in graph widgets (`2025-07`)

CloudWatch dashboard graph widgets support search expressions.

## Synthetics browser selection (`2025-09`)

Canary constructs expose the browser type.

## Synthetics canary groups (`2026-04`)

Synthetics constructs support canary groups.

## Synthetics Node.js 3.1 (`2026-03`)

The Synthetics canary runtime catalog includes Node.js 3.1.

## Synthetics Playwright 2.0 (`2025-06`)

The Synthetics runtime catalog includes Playwright 2.0.

## Synthetics run retries (`2025-07`)

Synthetics canaries accept `maxRetries` to configure automatic retries of canary runs.

## Synthetics tag replication (`2025-07`)

Synthetics constructs support tag replication.

## Token-safe Backup and Lambda validation (`2026-07`)

Backup lifecycle and vault-lock durations can be tokenized. Lambda validation also accepts tokenized provisioned-concurrency and asynchronous-invoke configuration values.
