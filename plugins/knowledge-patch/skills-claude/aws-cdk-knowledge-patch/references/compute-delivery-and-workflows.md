# Compute Delivery And Workflows

Lambda, Step Functions, Batch, CodeBuild, CodePipeline, Pipelines, and build assets. Items are organized by topic; parenthetical identifiers preserve exact extraction-batch attribution.

## ADOT Lambda layers (`2025-02`)

The Lambda ADOT layer catalog includes version 0.115.0.

## Attribute-based CodeBuild fleets (`2025-02`)

CodeBuild Fleet constructs support attribute-based compute types.

## Batch AL2023 images and default (`2026-04`)

Batch adds Amazon Linux 2023 image types and, under its feature flag, defaults to AL2023.

## Batch default instance classes (`2025-10`)

EC2 managed compute environments support default instance classes, and `useOptimalInstanceClasses` is deprecated.

## Batch job-definition updates (`2026-04`)

Batch skips unregistering a job definition during an update.

## Batch optimal instance classes remain supported (`2026-01`)

`useOptimalInstanceClasses` was undeprecated, reversing its earlier deprecation; it remains supported for Batch compute environments.

## Bun lockfile behavior (`2025-03`)

Lambda Node.js bundling no longer requires a frozen lockfile when Bun is used.

## Capacity-provider strategies for EcsRunTask (`2026-01`)

Step Functions `EcsRunTask` integrations for both Fargate and EC2 support capacity-provider strategies.

## CodeBuild fleet configuration (`2025-10`)

CodeBuild Fleets support custom instance types, VPC configuration, and overflow behavior.

## CodeBuild macOS 15 runners (`2025-12`)

CodeBuild constructs support macOS 15 runners.

## CodeBuild macOS 26 runners (`2026-03`)

CodeBuild constructs support macOS 26 runners.

## CodePipeline commands action (`2025-02`)

CodePipeline actions can run commands directly through a commands action.

## CodePipeline Git-push filters (`2025-03`)

CodePipeline L2 Git-push filters support branch and file criteria.

## CodePipeline stage conditions (`2025-03`)

CodePipeline L2 constructs support stage-level conditions.

## CodePipeline V2 in pipelines (`2025-04`)

The pipelines L3 construct supports the `V2` pipeline type.

## Combined CodePipeline trigger filters (`2025-05`)

CodePipeline configurations can use both `pullRequestFilter` and `pushFilter`.

## Consolidated Lambda integration permissions (`2025-11`)

REST and HTTP API Lambda integrations can opt to consolidate their Lambda permissions.

## Custom CSV delimiters (`2025-02`)

Step Functions `S3CsvItemReader` supports custom CSV delimiters.

## Deprecated Lambda policy feature flag (`2025-04`)

The default `@aws-cdk/aws-lambda:createNewPoliciesWithAddToRolePolicy` feature flag is deprecated.

## Deprecated Lambda runtime (`2025-01`)

The Lambda Python 3.8 runtime is marked deprecated.

## Distributed Map permissions (`2025-09`)

State machines synthesize the permissions needed to run and redrive Distributed Map, including maps defined only in nested `StateGraph` objects.

## Distributed Map result-writer configuration (`2025-04`)

Step Functions Distributed Map supports custom `WriterConfig` fields for its `ResultWriter`.

## Docker 27.4 tarball assets (`2025-04`)

`TarballImageAsset` supports the output format produced by Docker 27.4 and later.

## Docker build controls (`2025-09`)

`DockerBuildOptions` accepts a network parameter, and `TarballImageAsset` honors `CDK_DOCKER`.

## Dynamic Distributed Map result buckets (`2025-05`)

Step Functions `ResultWriter` accepts JSONPath or JSONata expressions for its bucket.

## Dynamic Step Functions queue ARNs (`2025-03`)

Step Functions task integrations allow `jobQueueArn` to be supplied with either JsonPath or JSONata.

## EvaluateExpression architecture (`2025-11`)

The Step Functions `EvaluateExpression` task supports selecting an architecture.

## Infinite Lambda event-source retries (`2025-04`)

Lambda `EventSourceMapping` accepts `retryAttempts: -1` to request infinite retries.

## Intrinsic Step Functions API endpoints (`2025-10`)

Under its feature flag, Step Functions tasks accept an intrinsic function as `apiEndpoint`.

## JSONata Map concurrency (`2026-01`)

Step Functions Map states accept JSONata expressions for `maxConcurrency`.

## JSONata Map item selectors (`2025-06`)

Step Functions Map states accept JSONata expressions in `ItemSelector`.

## Lambda capacity providers (`2025-12`)

Lambda constructs support capacity providers.

## Lambda durable functions (`2025-12`)

Lambda constructs support durable functions.

## Lambda Function URL origin addressing (`2025-10`)

CloudFront Lambda Function URL origins accept an `ipAddressType`.

## Lambda log removal policies (`2025-07`)

Lambda function constructs support setting a removal policy for their logs.

## Lambda multi-tenancy (`2025-11`)

Lambda constructs support multi-tenancy through `TenancyConfig`.

## Lambda Node.js 24 defaults (`2026-06`)

Lambda framework functions and custom resources now default to `nodejs24.x`, and `Runtime.NODEJS_LATEST` resolves to it in every region. Node.js 24 does not support callback-style asynchronous handlers; migrate them to `async` handlers or pin `Runtime.NODEJS_22_X` (or set `useLatestRuntimeVersion: false` on `NodejsFunction`).

## Lambda Node.js parent-path entries (`2026-06`)

Lambda Node.js bundling accepts entry paths containing `..`.

## Lambda Ruby 3.4 (`2025-03`)

The Lambda runtime catalog includes Ruby 3.4.

## Lambda Ruby 4.0 (`2026-04`)

The Lambda runtime catalog includes Ruby 4.0.

## Lambda tag propagation (`2025-06`)

Lambda functions can propagate their tags to their log groups.

## Lambda-authorizer roles (`2026-04`)

API Gateway v2 Lambda authorizers support an explicitly configured role.

## Latest Lambda Node.js fallback (`2025-10`)

The fallback used for the latest Lambda Node.js runtime is now Node.js 22.x.

## Managed Lambda log-group flag default (`2025-06`)

When `aws-lambda:useCdkManagedLogGroup` is absent, CDK treats the feature flag as disabled.

## Multi-value headers for Lambda target groups (`2025-05`)

Elastic Load Balancing v2 Lambda target groups support multi-value headers.

## New Bun lockfile support (`2025-09`)

CDK recognizes the newer Bun lockfile format.

## Node.js 22 expression evaluation (`2025-09`)

Step Functions `EvaluateExpression` supports Node.js 22.

## Parallel-state parameters (`2025-05`)

Step Functions `Parallel` states support parameters.

## Pipeline CodeBuild configuration propagation (`2025-12`)

CDK Pipelines now propagates CodeBuild `fleet` and `certificate` settings instead of dropping them from the generated projects.

## Pipeline invoke actions (`2025-04`)

CodePipeline action constructs support invoking a pipeline.

## Pipeline manual-approval metadata (`2025-06`)

CDK Pipelines manual approvals can include a review URL and an SNS notification topic.

## Pipeline service roles for actions (`2025-04`)

CodePipeline L2 adds `usePipelineRoleForActions`, and pipelines actions can default to the pipeline service role instead of creating a separate role.

## Remote Docker servers (`2025-09`)

CodeBuild supports remote Docker servers, and pipelines' `CodeBuildFactory` can use Docker-server support.

## Shared CodeBuild caches (`2025-07`)

CodeBuild project constructs support cache sharing.

## Step Functions JSONata and variables (`2025-02`)

Step Functions constructs support JSONata and workflow variables.

## Windows Server Core 2022 build images (`2025-08`)

CodeBuild supports Windows Server Core 2022 images with on-demand capacity.
