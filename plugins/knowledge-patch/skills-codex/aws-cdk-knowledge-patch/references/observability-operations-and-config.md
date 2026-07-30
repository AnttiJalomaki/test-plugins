# Observability, operations, and configuration

CloudWatch, CloudWatch Logs, Synthetics, AppConfig, Backup, SES, regional catalogs, and operational service constructs.

## AppConfig

### AppConfig configuration-profile deletion protection

*Batch 2025-05.*

AppConfig configuration profiles now participate in deletion-protection checks.

### AppConfig deployment-tick extensions

*Batch 2025-01.*

AppConfig L2 extensions support the `atDeploymentTick` action point.

### AppConfig environment deletion protection

*Batch 2025-01.*

AppConfig environments can opt into deletion protection.

### KMS-encrypted AppConfig hosted configurations

*Batch 2025-06.*

AppConfig hosted configurations support encryption with a customer-managed key.

## Backup

### Backup schedule time zones

*Batch 2025-07.*

Backup constructs support `ScheduleExpressionTimezone`.

### Token-safe Backup and Lambda validation

*Batch 2026-07.*

Backup lifecycle and vault-lock durations can be tokenized. Lambda validation also accepts tokenized provisioned-concurrency and asynchronous-invoke configuration values.

## CloudWatch and CloudWatch Logs

### CloudWatch anomaly-detection alarms

*Batch 2025-05.*

CloudWatch constructs support anomaly-detection alarms.

### CloudWatch Logs deletion protection

*Batch 2026-01.*

CloudWatch Logs constructs support deletion-protection configuration.

### CloudWatch Logs destination-policy typing

*Batch 2025-05.*

The `DeliveryDestinationPolicy` property of `AWS::Logs::DeliveryDestination` now uses the `DestinationPolicy` type instead of arbitrary JSON.

### CloudWatch Logs field indexes

*Batch 2025-03.*

The Log Group L2 construct accepts `fieldIndexPolicies`.

### CloudWatch Logs transformers

*Batch 2025-07.*

CloudWatch Logs constructs support log transformers.

### CloudWatch metric identity and visibility

*Batch 2025-07.*

CloudWatch `Metric` exposes `id` and `visible` properties.

### CloudWatch pie-chart labels

*Batch 2025-09.*

CloudWatch pie charts can display labels.

### CloudWatch quoted metric-math strings

*Batch 2026-06.*

CloudWatch metric-math validation no longer treats quoted strings as unknown identifiers.

### Cross-account CloudWatch widgets

*Batch 2025-07.*

CloudWatch log-query and metric widgets accept an account ID for cross-account visibility.

### Dashboard query languages

*Batch 2025-07.*

CloudWatch dashboard constructs expose the `queryLanguage` property.

### Infrequent Access logs in ADC regions

*Batch 2025-07.*

The CloudWatch Logs Infrequent Access log class is supported in ADC regions.

### Lambda tag propagation

*Batch 2025-06.*

Lambda functions can propagate their tags to their log groups.

### Metric filters on transformed logs

*Batch 2025-10.*

CloudWatch Logs metric filters can opt to run against transformed logs.

### Metric-filter dimension exposure

*Batch 2025-06, 2025-07.*

Metrics exposed from CloudWatch Logs metric filters briefly retained the filter's dimension map, but that behavior was reverted; exposed metrics no longer retain the dimension map.

### Multiple stats commands in log queries

*Batch 2025-07.*

CloudWatch Logs query strings can contain multiple `stats` commands.

### PromQL alarms

*Batch 2026-05.*

CloudWatch provides a PromQL Alarm L2 construct.

### Regex JSON metric filters

*Batch 2025-02.*

CloudWatch Logs JSON metric filters support regular-expression patterns.

### Search expressions in graph widgets

*Batch 2025-07.*

CloudWatch dashboard graph widgets support search expressions.

## Regional and service catalogs

### MediaConnect L2 constructs

*Batch 2026-07.*

MediaConnect now has L2 construct support.

### MediaPackage v2 region and name handling

*Batch 2026-04.*

MediaPackage v2 resources expose a region attribute and apply additional naming validation.

### Region Info additions

*Batch 2025-07.*

Region Info supports the `eusc-de` and `ap-southeast-6` regions.

### Typed Glue partition projection

*Batch 2026-02.*

Glue constructs support typed partition-projection configuration.

## SES

### SES configuration-set email validation

*Batch 2026-05.*

SES configuration sets support automatic email validation.

### SES custom-tracking HTTPS policy

*Batch 2025-05.*

SES constructs support an HTTPS policy for custom tracking domains.

## Synthetics

### Built-in ECS linear and canary deployments

*Batch 2026-01.*

ECS constructs now provide built-in linear and canary deployment configurations.

### New Synthetics Python runtimes

*Batch 2025-05.*

The Synthetics runtime catalog includes Python canary runtimes 5.0 and 5.1.

### New Synthetics runtimes

*Batch 2025-01.*

The Synthetics runtime catalog now includes Node Playwright 1.0 and Python Selenium 4.1.

### New Synthetics runtimes

*Batch 2026-01.*

The Synthetics runtime catalog adds `syn-nodejs-3.0`, Playwright 4.0 and 5.0, and Puppeteer 12.0 and 13.0.

### New Synthetics runtimes

*Batch 2026-05.*

The Synthetics runtime catalog includes Playwright 5.1 and 6.0.

### Root-level Synthetics scripts

*Batch 2025-10.*

Synthetics canary assets may place script files at the archive root when using Puppeteer runtime version 11 or later.

### Safe canary updates

*Batch 2025-06.*

Synthetics constructs support safe canary updates.

### Synthetics browser selection

*Batch 2025-09.*

Canary constructs expose the browser type.

### Synthetics canary groups

*Batch 2026-04.*

Synthetics constructs support canary groups.

### Synthetics Node.js 3.1

*Batch 2026-03.*

The Synthetics canary runtime catalog includes Node.js 3.1.

### Synthetics Playwright 2.0

*Batch 2025-06.*

The Synthetics runtime catalog includes Playwright 2.0.

### Synthetics run retries

*Batch 2025-07.*

Synthetics canaries accept `maxRetries` to configure automatic retries of canary runs.

### Synthetics tag replication

*Batch 2025-07.*

Synthetics constructs support tag replication.
