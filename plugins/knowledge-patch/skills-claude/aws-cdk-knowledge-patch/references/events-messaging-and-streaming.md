# Events Messaging And Streaming

EventBridge, Scheduler, Kinesis, Firehose, SNS, SQS, Kafka, MSK, and IoT. Items are organized by topic; parenthetical identifiers preserve exact extraction-batch attribution.

## API destination policy ARN (`2025-07`)

EventBridge API destinations expose an `arnForPolicy` attribute.

## API Gateway v2 SQS integrations (`2025-02`)

API Gateway v2 integration constructs support SQS.

## Batched IoT HTTP actions (`2026-01`)

IoT `HttpAction` supports batching messages. `enableBatchConfig` is explicitly disabled by default, so batching remains opt-in.

## Encrypted SNS notification policies (`2025-05`)

Under its feature flag, S3 notifications to a KMS-encrypted SNS topic add a key policy that trusts S3.

## EventBridge HTTP integration defaults (`2026-05`)

`HttpEventBridgeIntegration` automatically includes `EventBusName` in its default parameter mapping.

## EventBridge logging and archive encryption (`2025-09`)

Event buses support logging configuration, and `Archive` can use customer-managed keys.

## EventBridge PutEvents HTTP API integration (`2026-01`)

API Gateway v2 integrations can invoke EventBridge `PutEvents`.

## EventBridge rule roles (`2025-04`)

EventBridge `Rule` constructs support an explicitly configured role.

## Firehose destination time zones (`2025-07`)

Kinesis Data Firehose S3 destinations support custom time-zone settings.

## Firehose destinations for EC2 flow logs (`2026-02`)

EC2 flow-log destinations accept Firehose `IDeliveryStreamRef` values.

## Firehose dynamic partitioning (`2026-02`)

Kinesis Data Firehose constructs support dynamic partitioning.

## Firehose integrations and processors (`2025-11`)

EventBridge Data Firehose targets accept Firehose's `IDeliveryStream`. Delivery streams also provide built-in processors for decompressing CloudWatch Logs data and extracting messages.

## Firehose output integrations (`2025-04`)

Kinesis Data Firehose supports S3 file-extension formats, and CloudWatch Logs destination constructs can target Amazon Data Firehose.

## Firehose record-format conversion (`2025-10`)

Kinesis Data Firehose `DeliveryStream` constructs support record-format conversion for S3 bucket destinations.

## Firehose SNS subscriptions (`2025-06`)

SNS subscription constructs support Amazon Data Firehose destinations.

## High-throughput FIFO topics (`2025-02`)

SNS constructs support high-throughput mode for FIFO topics.

## HTTP APIs as EventBridge targets (`2025-04`)

EventBridge target constructs support API Gateway v2 `HttpApi`.

## Kafka event-source failure destinations (`2025-11`)

Lambda Kafka event-source mappings support an on-failure destination.

## Kafka event-source observability (`2026-02`)

Lambda Kafka event-source mappings support observability configuration.

## Kafka schema registries for Lambda (`2025-06`)

Lambda Kafka event-source constructs support schema-registry configuration.

## Kinesis Analytics v2 package (`2025-09`)

Using Kinesis Analytics v2 through `aws-kinesisanalytics` is deprecated; use `aws-kinesisanalyticsv2`.

## Kinesis shard-level metrics (`2025-10`)

Kinesis stream constructs expose shard-level metrics.

## Kinesis stream consumers (`2025-02`)

Kinesis constructs support stream consumers.

## L2 event patterns on L1 rules (`2025-11`)

EventBridge L2 `EventPattern` interfaces can be used with `CfnRule`.

## Message groups on standard SQS targets (`2025-12`)

EventBridge SQS targets support `messageGroupId` for standard queues as well as FIFO queues.

## MSK Express brokers (`2025-11`)

MSK constructs support Express brokers.

## SNS EventBridge targets with IAM roles (`2025-05`)

The EventBridge `SnsTopic` target can opt into using an IAM role.

## SQS provisioned pollers (`2026-05`)

Lambda SQS event-source mappings support `provisionedPollerConfig`, including validation and corrected typing.

## Stable EventBridge Scheduler (`2025-03`)

EventBridge Scheduler and its target constructs graduated from experimental to stable. Scheduler targets also include `EcsRunTask`.

## Stable Firehose constructs (`2025-02`)

Kinesis Data Firehose constructs graduated from experimental to stable.

## Timestamp starts for Kafka event sources (`2025-03`)

Lambda Kafka event sources support a starting-position timestamp.
