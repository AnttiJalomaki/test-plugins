---
name: aws-cdk-knowledge-patch
description: AWS CDK
version: 2.262.1
license: MIT
metadata:
  author: Nevaberry
---


# AWS CDK Knowledge Patch

Use this skill when designing, upgrading, synthesizing, testing, or reviewing
AWS CDK applications. Start with the compatibility checks below, then open the
topic reference that owns the construct or service being changed.

## Working method

1. Read the project manifest and lockfile to identify the installed CDK
   packages and feature-flag posture.
2. Check breaking contracts, removed generated properties, changed defaults,
   deprecated spellings, and runtime migrations before adding new features.
3. Open the relevant topic reference and retain any exact batch attribution
   needed to reason about intermediate behavior.
4. Prefer the installed declarations, synthesized template, application tests,
   and CloudFormation behavior when they disagree with an older assumption.
5. Run synthesis and validation, inspect the template diff, and test destructive
   lifecycle, IAM, cross-account, and replacement behavior explicitly.

## Reference index

| Reference | Topics |
| --- | --- |
| [core-toolkit-and-validation.md](references/core-toolkit-and-validation.md) | Toolkit and CLI contracts, synthesis, context, custom resources, validation, feature flags, assets, and core APIs |
| [generated-contracts-and-types.md](references/generated-contracts-and-types.md) | Required and removed L1 properties, immutable fields, generated imports and guards, reference interfaces, and type narrowing |
| [networking-apis-and-edge.md](references/networking-apis-and-edge.md) | API Gateway, CloudFront, Route 53, load balancers, EC2, VPC, Auto Scaling, endpoints, and certificates |
| [containers-and-kubernetes.md](references/containers-and-kubernetes.md) | ECS, EKS, ECR, cluster lifecycle, capacity providers, deployment controls, AMIs, and Kubernetes integration |
| [compute-delivery-and-workflows.md](references/compute-delivery-and-workflows.md) | Lambda, Step Functions, Batch, CodeBuild, CodePipeline, Pipelines, runtimes, bundling, and workflow integrations |
| [storage-and-databases.md](references/storage-and-databases.md) | S3, DynamoDB, RDS, Aurora, OpenSearch, DocumentDB, EFS, Glue, ElastiCache, replication, encryption, and lifecycle |
| [events-messaging-and-streaming.md](references/events-messaging-and-streaming.md) | EventBridge, Scheduler, Kinesis, Firehose, SNS, SQS, Kafka, MSK, IoT, targets, processors, and event sources |
| [identity-security-and-ai.md](references/identity-security-and-ai.md) | IAM, KMS, Cognito, secrets, grants, Bedrock, AgentCore, SageMaker, SES, trust, and authorization |
| [observability-and-service-integrations.md](references/observability-and-service-integrations.md) | CloudWatch, Logs, Synthetics, AppConfig, AppSync, Backup, EMR, dashboards, canaries, and service integrations |

## Breaking changes and changed defaults

### Treat validation as a public contract

- Many construct libraries and the CLI now throw typed `ValidationError`
  instances. Catch by type or error code; do not parse message text.
- CDK errors and annotations can carry error codes and source or external
  traces. Preserve these fields when wrapping or reporting failures.
- Apps can register validation plugins through `Validations`, using
  `addWarning`, `addError`, and `acknowledge`; policy-validation interfaces no
  longer use the `policyValidationBeta1` suffix.
- Validation reports are always emitted in the cloud assembly, include
  construct annotations and suppressed violations, and use the newer
  self-contained schema. Update report consumers before relying on old fields.
- `failSynthOnValidationErrors` can suppress the validation exit code and
  console output. It does not mean the report contains no errors.
- Built-in template validation runs by default. For a deliberate diagnostic
  bypass, set `CDK_VALIDATION=false` for that invocation:

```sh
CDK_VALIDATION=false cdk synth
```

### Audit generated L1 contracts before compiling or deploying

- CloudFormation schema updates can add required properties, remove attributes
  and nested types, narrow value types, or mark fields immutable.
- An immutable change means replacement, even if the same property previously
  updated in place. Inspect the diff for stateful resources and fleet changes.
- Do not depend on a generic `Id` attribute. Several generated resources have
  removed it, and some now expose a named primary identifier instead.
- Generated resources support import factories and static resource type guards.
  Prefer these to ad hoc attribute reconstruction or `instanceof` assumptions.
- Known L1 relationships can accept construct objects, but public APIs may now
  expose narrow reference interfaces. Type-test before accessing richer L2
  members.
- `IEncryptedResource` is environment-aware but no longer necessarily an
  `IResource`; require the intersection or guard with `Resource.isResource()`
  when resource behavior is needed.

### Migrate runtime defaults deliberately

- Framework functions, custom resources, and `Runtime.NODEJS_LATEST` now select
  Node.js 24. Callback-style asynchronous handlers are unsupported there.
- Convert handlers to `async`, or pin `Runtime.NODEJS_22_X`; for
  `NodejsFunction`, `useLatestRuntimeVersion: false` is also an escape hatch.
- The Lambda catalog includes newer Node.js, Python, Java, and Ruby runtimes;
  Python 3.8 is deprecated.
- Python custom resources use Python 3.13, avoiding creation of an end-of-life
  Python 3.9 rotation function.
- CloudFront Functions can default to JavaScript 2.0 under its feature flag.
  Test runtime-specific syntax before enabling the flag.

### Respect explicit reversals and staged behavior

- Metrics returned from CloudWatch Logs metric filters do not retain the
  filter's dimension map; a short-lived change that added it was reverted.
- EKS kubectl subnets that are isolated now produce a warning, not the earlier
  error. Still verify that kubectl connectivity is actually possible.
- Batch `useOptimalInstanceClasses` remains supported; its earlier deprecation
  was reversed.
- Native ECS blue/green L2 support was briefly reverted and then restored.
  Verify the installed declarations instead of inferring availability from one
  intermediate release.
- For `AWS::ECS::Service`, `AvailabilityZoneRebalancing` defaults to
  `DISABLED`. Set the property explicitly when rebalancing is required.

### Update renamed and semantic APIs

- Use RDS `clusterScalabilityType`, not the erroneous
  `clusterScailabilityType`.
- `FluentdLogDriver` uses `async`; `asyncConnect` is deprecated.
- `Source.jsonData()` does not escape JSON automatically. Pass
  `{ escape: true }` when the previous escaping behavior is required.
- KMS `Alias` references resolve to the alias, not the backing key. Imported
  aliases expose `aliasTargetKey` for code that truly needs the key.
- Kinesis Analytics v2 use through `aws-kinesisanalytics` is deprecated; use
  `aws-kinesisanalyticsv2`.
- The public CLI plugin contract is the supported plugin boundary. Do not
  import internal CLI libraries.

## High-value core and synthesis capabilities

- `cdk bootstrap --untrust` retracts bootstrap trust, and the CLI supports
  simplified CloudFormation resource import.
- `CDK_TOOLKIT_VERSION` is a supported cloud-assembly environment contract.
- Cloud assemblies expose feature-flag configuration and information to
  downstream consumers.
- Context lookups can use an additional cache key to separate otherwise
  identical requests.
- Property injectors apply across L2 constructors. Mixins provide another
  stable extension mechanism, with Aspect conversion helpers and property
  merge strategies for objects, arrays, and deferred `Box` values.
- `Box` preserves source traces for deferred values; L1 property mutations also
  record source traces for post-construction diagnostics.
- Weak cross-stack references work within and across environments, including
  list-valued attributes. Use `Fn::GetStackOutput` for cross-region outputs.
- `RemovalPolicies.of(scope)` applies lifecycle policy across a scope. ECR and
  S3 also provide auto-delete mixins.
- Git source metadata can be included in synthesized templates.

## Service design checkpoints

### Networking, APIs, and edge

- API Gateway supports dual-stack REST, HTTP, WebSocket, and domain
  configurations; domain policies can use TLS 1.3.
- Private APIs accept resource policies. HTTP stages support access logs, and
  WebSocket stages support variables, access logs, usage plans, and API keys.
- Lambda integrations can consolidate permissions, authorizers can use an
  explicit role, and APIs can stream responses with a configured transfer mode.
- CloudFront supports VPC and gRPC origins, origin-group selection, versioned
  reads, origin IP controls, and response-completion timeouts.
- Network Load Balancers support subnet mappings and feature-flagged default
  security-group settings. Application Load Balancers support JWT verification.
- Route 53 supports SVCB, HTTPS, and failover records, plus accelerated
  recovery for public hosted zones. Supplying TTL with an alias emits a
  diagnostic.

### Containers and Kubernetes

- Experimental EKS clusters require a `kubectlLayer` that matches the
  Kubernetes version; the EKS v2 construct set is stable.
- EKS supports Hybrid Nodes, native OIDC providers, managed node repair,
  additional access-entry types, service-account overwrite control, deletion
  protection, and removal policies across constructs.
- Feature flags can select recommended AL2023 EKS AMIs and AL2023 Batch images.
- ECS supports native blue/green, linear, and canary deployments; forced
  deployments; fault injection; enhanced Container Insights; Service Connect
  TLS and access logs; and managed-storage encryption.
- Managed Instances capacity providers create their instance profile, require
  at least one security group, and can choose Spot capacity.
- ECR supports tag-mutability exclusions, existing-repository lookup, Docker
  build contexts, and an auto-delete-images mixin.

### Compute, workflows, and delivery

- Lambda supports durable functions, capacity providers, multi-tenancy,
  Kafka observability and failure destinations, SQS provisioned pollers, and S3
  destinations for failed stream records.
- Event source mappings accept infinite retry with `retryAttempts: -1`, but use
  it only with a bounded failure-handling design.
- Step Functions supports JSONata, workflow variables, dynamic ARNs, dynamic
  result buckets, item selectors and concurrency, richer Distributed Map result
  writers, and generated run/redrive permissions.
- CodePipeline supports commands, stage conditions, combined push and pull
  request filters, pipeline invocation, EC2 deployment, ECR publishing, and
  Inspector scans.
- Pipelines can use V2 pipelines, propagate CodeBuild fleet and certificate
  settings, configure `cdk-assets`, and attach manual-approval review metadata.
- CodeBuild supports shared caches, remote Docker servers, richer fleets,
  macOS runners, Windows Server Core images, and attribute-based compute.

### Storage and databases

- S3 supports object replication and filters, custom replication roles,
  attribute-based access control, blocked encryption types, bucket name
  prefixes and namespaces, and the S3 Files Lambda L1 integration.
- Feature-flagged S3 public-access defaults enable settings whose defaults are
  true. Treat that flag as a security migration.
- DynamoDB supports MRSC and cross-account global-table replication, compound
  GSI keys, stream resource policies, contributor-insights modes, and correct
  index permissions when grants precede index creation.
- RDS supports deferred modifications, lifecycle configuration, native Secrets
  Manager credentials, proxy endpoints and authentication schemes, standalone
  parameter groups, snapshot restore paths, and retained automated backups.
- Aurora supports Database Insights, Limitless, Serverless v2 auto-pause, and
  explicit instance Availability Zones.
- OpenSearch defaults to TLS 1.2, honors automatic-update opt-out, recognizes
  local-storage instance families, supports S3 Vectors, and accepts larger gp3
  throughput values.

### Events, identity, AI, and operations

- Stable Scheduler targets include `EcsRunTask`. EventBridge supports logging,
  encrypted archives, HTTP API targets and integrations, rule roles, richer
  SQS/SNS targets, and Firehose delivery streams.
- Stable Firehose constructs support record conversion, dynamic partitioning,
  S3 time zones and extensions, CloudWatch Logs processors, and SNS
  subscriptions.
- Cognito supports managed login, choice-based and passkey authentication,
  refresh-token rotation, analytics, and the newer pre-token trigger event.
- Use native IAM and EKS OIDC providers when avoiding custom-resource-backed
  providers. Imported KMS aliases can grant under their feature flag.
- Bedrock AgentCore constructs are stable and cover runtimes, tools, memory,
  authorizers, endpoints, and gateway IAM credential targets.
- CloudWatch supports anomaly detection, PromQL alarms, query languages,
  cross-account widgets, search expressions, and richer metric identity.
- Synthetics adds current Playwright, Puppeteer, Node.js, Python, browser,
  retry, group, tag-replication, safe-update, and root-script capabilities.

## Verification checklist

- Confirm package versions and feature flags from manifests and lockfiles.
- Compile against installed generated declarations after any upgrade.
- Run `cdk synth` with validation enabled and inspect validation reports.
- Review `cdk diff` for replacement, removal-policy, encryption, and IAM
  changes.
- Test tokenized values in validation-sensitive properties.
- Exercise cross-account, cross-region, imported-resource, and weak-reference
  paths that the stack uses.
- Verify runtime handlers on the selected Lambda or custom-resource runtime.
- Check generated policies for indexes, encrypted notifications, deployment
  resources, Distributed Map, and integrations.
- Open the relevant reference for the complete service-specific behavior list
  before finalizing the change.
