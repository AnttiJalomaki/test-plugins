---
name: aws-cdk-knowledge-patch
description: AWS CDK
version: 2.262.1
license: MIT
metadata:
  author: Nevaberry
---


# AWS CDK Knowledge Patch

Use this skill when writing, reviewing, upgrading, or debugging AWS CDK
applications. Start with the breaking changes and defaults below, then open the
reference that matches the service or workflow in question.

## Reference index

| Reference | Topics |
| --- | --- |
| [Core, CLI, synthesis, and validation](references/core-cli-and-synthesis.md) | Toolkit contracts, validation plugins and reports, custom resources, synthesis, core APIs, and cross-stack references |
| [Generated L1 and CloudFormation contracts](references/generated-l1-contracts.md) | Required, removed, and immutable L1 fields; generated imports, interfaces, type guards, and property tracing |
| [Compute, containers, and runtimes](references/compute-containers-and-runtime.md) | Lambda, ECS, EKS, EC2, Auto Scaling, Batch, EMR, deployment strategies, and runtime defaults |
| [Storage, databases, and search](references/storage-and-databases.md) | S3, S3 Tables, DynamoDB, RDS, Aurora, ECR, EFS, DocumentDB, ElastiCache, and OpenSearch |
| [Networking, edge, APIs, and DNS](references/networking-edge-and-dns.md) | API Gateway, CloudFront, load balancers, VPC networking, Route 53, and ACM |
| [Events, messaging, and workflows](references/events-messaging-and-workflows.md) | Step Functions, EventBridge, Kinesis, Data Firehose, SNS, SQS, Kafka, MSK, AppSync, and IoT |
| [Identity, security, and AI services](references/identity-security-and-ai.md) | IAM, KMS, Cognito, Secrets Manager, Bedrock, AgentCore, SageMaker, and grants |
| [Observability, operations, and configuration](references/observability-operations-and-config.md) | CloudWatch, CloudWatch Logs, Synthetics, AppConfig, Backup, SES, and service catalogs |
| [Delivery, assets, and pipelines](references/delivery-assets-and-pipelines.md) | CDK Pipelines, CodePipeline, CodeBuild, Docker, bundling, staging, and source packaging |

## Working method

1. Read the project's manifest to determine its pinned `aws-cdk-lib` and CLI
   versions. Apply only guidance available to that version.
2. Identify whether the code uses generated L1 resources, stable L2 constructs,
   experimental packages, or an L3 pattern; their compatibility risks differ.
3. Check feature flags in `cdk.json` and the synthesized cloud assembly before
   relying on a new default.
4. Open the narrowest reference file for the service being changed. Cross-service
   integrations may require two references.
5. Run synthesis and inspect the template, validation report, IAM policies,
   replacements, and runtime settings before deployment.

## Breaking changes and defaults

### Built-in template validation is comprehensive

Built-in template validation now runs a comprehensive default rule set.
Temporarily disable it for one invocation with:

```sh
CDK_VALIDATION=false cdk synth
```

Prefer fixing or explicitly acknowledging violations. Validation reports are
always written to the cloud assembly, include construct annotations and
suppressed violations, and are self-contained. Report consumers must accept the
new schema and must not assume every listed violation failed synthesis.

### Validation failures are typed and coded

Construct validation across core and many service libraries throws
`ValidationError` rather than generic errors. CLI callers can distinguish error
classes without parsing messages, and errors and annotations can carry error
codes. `ConstructError` can include external traces, which are also appended to
cloud-assembly metadata when available.

### Generated L1 contracts require upgrade review

Generated CloudFormation bindings have accumulated required fields, immutable
properties, removed attributes, renamed types, and fields whose update now
replaces a resource. Before upgrading:

- Compile every language binding that directly consumes `Cfn*` properties or
  attributes.
- Diff synthesized templates for replacement-causing changes.
- Replace reads of removed `Id` attributes with the service's supported
  identifier or reference interface.
- Review the generated-contract reference for the exact affected resources.

Every generated L1 also exposes `from<Resource>Arn` and
`from<Resource><Prop>` import factories, shared L1/L2 resource interfaces, and a
static `isCfn<ResourceName>` type guard:

```ts
if (s3.CfnBucket.isCfnBucket(value)) {
  // value is a CfnBucket
}
```

### Lambda defaults use Node.js 24

Lambda framework functions and custom resources default to `nodejs24.x`, and
`Runtime.NODEJS_LATEST` resolves to it in every region. Node.js 24 rejects
callback-style asynchronous handlers; convert them to `async`, pin
`Runtime.NODEJS_22_X`, or set `useLatestRuntimeVersion: false` on
`NodejsFunction`.

Earlier regional custom-resource defaults are therefore historical when using
the newer release. Also note that asynchronous custom-resource provider logging
defaults to off, and the managed Lambda log-group feature flag remains disabled
when absent.

### Several exposed types are intentionally narrower

Job queues, backup rules, event destinations, log-group results, and API
destination imports may now expose reference interfaces instead of richer L2
interfaces. Type-test or cast only when richer members are genuinely required.

`IEncryptedResource` extends `IEnvironmentAware`, not `IResource`. Code needing
both contracts should use `IEncryptedResource & IResource` or guard with
`Resource.isResource()`. EC2 `IPeer` methods now return specific interfaces
instead of `any`.

### `Source.jsonData()` escaping is opt-in

S3 Deployment no longer automatically escapes JSON written by
`Source.jsonData()`. Request the former behavior when special characters need
it:

```ts
Source.jsonData("config.json", data, { escape: true })
```

Tokens inside lists are resolved, and `Source.data()` accepts an empty string.

### KMS alias references preserve alias identity

`Alias` references resolve to the alias rather than the underlying key.
Aliases imported with `Alias.fromAliasName()` expose `aliasTargetKey`; grant
methods on imported aliases require the corresponding feature flag. Audit code
that assumed an alias token resolved to a key ARN.

### Service defaults and reversions matter

- `AWS::ECS::Service.AvailabilityZoneRebalancing` defaults to `DISABLED`,
  despite earlier L2 support for availability-zone rebalancing.
- CloudWatch Logs metric-filter metrics no longer retain the filter's
  dimension map; the short-lived retention behavior was reverted.
- Batch `useOptimalInstanceClasses` remains supported after its deprecation was
  reversed.
- The EKS isolated-kubectl-subnet diagnostic is a warning, not an error.
- Under their feature flags, Batch and EKS can default to Amazon Linux 2023,
  and CloudFront Functions can default to JavaScript 2.0.
- OpenSearch domains default to the TLS 1.2 security policy.

### Deprecations and removals

- Use `aws-kinesisanalyticsv2`; Kinesis Analytics v2 through
  `aws-kinesisanalytics` is deprecated.
- DynamoDB's point-in-time recovery specification replaces the older recovery
  setting.
- `ARecord`'s delete-existing field, ECS
  `canContainersAccessInstanceRole`, and Lambda Python 3.8 are deprecated.
- The default `@aws-cdk/aws-lambda:createNewPoliciesWithAddToRolePolicy`
  feature flag is deprecated.
- Legacy Bedrock foundation-model entries for Claude 2, Claude 2.1, and Claude
  Instant are deprecated.

## High-value new capabilities

### Validation plugins and source-aware diagnostics

Use `Validations` to register validation plugins and to call `addWarning`,
`addError`, and `acknowledge`. Policy-validation interfaces are stable without
the former `Beta1` suffix. Plugins receive `IPolicyValidationContext.scope`,
may create cloud-assembly files, and can have violations suppressed.

Core also provides source-preserving `Box` deferred values. L1 property
mutations record source traces, improving diagnostics for changes made after
construction.

### Mixins and bulk policy behavior

Mixins provide an extension mechanism alongside Aspects.
`@aws-cdk/cfn-property-mixins` is stable, Aspect/Mixin conversion helpers are
available, and `PropertyMergeStrategy` handles CloudFormation property objects,
array merge strategies, and deferred `Box` values. S3 and ECS service mixins
ship in `aws-cdk-lib`; S3 bucket and ECR repository auto-delete mixins are
available.

### Cross-stack references have safer options

Weak references work within and across environments, including list-valued
attributes, and avoid forcing unnecessary deployment ordering. `Fn::GetStackOutput`
supports cross-region references and avoids the earlier export-update failure.
Choose reference strength deliberately.

### Step Functions supports JSONata deeply

Workflows support JSONata and variables. Map states accept JSONata for
`ItemSelector` and `maxConcurrency`; Distributed Map result buckets and queue
ARNs can be dynamic, and synthesized policies include run and redrive
permissions even for nested `StateGraph` definitions.

### ECS and EKS deployment surfaces expanded

ECS supports native blue/green deployment through L1 and L2 constructs, plus
built-in linear and canary configurations. Managed Instances capacity providers
create their instance profile, require a security group, and support Spot
capacity.

EKS v2 constructs are stable. Current catalogs include Kubernetes 1.35,
additional access-entry types, native OIDC providers, broad removal-policy
support, cluster deletion protection, and service-account overwrite control.

### Data constructs cover newer architectures

DynamoDB `TableV2` supports MRSC and cross-account global-table replication.
S3 supports object replication, attribute-based access control, blocked
encryption types, name prefixes and namespaces, and Tables L2 constructs.
RDS supports native Secrets Manager credentials, Database Insights,
Aurora Serverless v2 auto-pause, proxy endpoints, and standalone parameter
groups.

### Networking and API integrations are broader

API Gateway supports dual-stack REST, HTTP, WebSocket, and domain resources,
TLS 1.3 domain policies, response streaming, private API resource policies, and
WebSocket usage plans. CloudFront supports VPC origins, gRPC, origin-group
selection criteria, host-header-only forwarding, and expanded HTTP-origin
controls. Application Load Balancers support JWT verification.

### Observability is construct-native

CloudWatch provides anomaly-detection and PromQL alarm constructs, dashboard
query languages, cross-account widgets, search expressions, and pie labels.
CloudWatch Logs adds field indexes, transformers, transformed-log metric
filters, deletion protection, regular-expression JSON filters, and multiple
`stats` commands. Synthetics adds browser selection, safe updates, groups,
retries, tag replication, and newer Playwright, Puppeteer, Python, and Node.js
runtimes.

## Verification checklist

- Compile against the pinned CDK libraries in every language binding used.
- Run `cdk synth` with the project's normal context and feature flags.
- Review validation output and `validation-report.json`.
- Diff CloudFormation properties, logical IDs, IAM grants, and replacement
  behavior.
- Confirm Lambda and custom-resource runtimes and handler style.
- Check generated asset, Docker, and pipeline project configuration.
- Verify service defaults rather than assuming an omitted property retains its
  historical behavior.
