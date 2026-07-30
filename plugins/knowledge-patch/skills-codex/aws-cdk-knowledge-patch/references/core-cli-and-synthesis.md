# Core, CLI, synthesis, and validation

Toolkit contracts, synthesis behavior, validation, custom resources, cross-stack references, and core extension APIs.

## CLI and toolkit contracts

### Bootstrap trust removal

*Batch 2025-01.*

`cdk bootstrap` accepts `--untrust`, providing a direct way to retract bootstrap trust.

### Public CLI plugin contract

*Batch 2025-01.*

A public CLI-plugin contract now defines the supported boundary between the CLI and plugins, and imports of internal CLI libraries are disallowed. Credential plugins may return `null` for expiration and may initially return empty credentials.

### Simplified resource import

*Batch 2025-01.*

The CLI supports CloudFormation simplified resource import, reducing the setup required for bringing existing resources under stack management.

### Toolkit version environment contract

*Batch 2025-03.*

The cloud-assembly API now declares `CDK_TOOLKIT_VERSION` as a supported environment variable.

## Core extension APIs

### CDK Mixins

*Batch 2026-03.*

CDK introduces Mixins as an extension mechanism, graduates `@aws-cdk/cfn-property-mixins` to stable, and provides helpers for converting between Aspects and Mixins. `PropertyMergeStrategy` can merge arbitrary CloudFormation property objects, and S3 and ECS service mixins now ship in `aws-cdk-lib`.

### Deferred values with source traces

*Batch 2026-04.*

Core provides a `Box` API for deferred values that preserves accurate stack traces.

### Property injection across L2 constructs

*Batch 2025-05.*

Property injectors can target all L2 constructors.

### Property merge strategies

*Batch 2026-05.*

`PropertyMergeStrategy` now supports array merge strategies, and the built-in strategies are compatible with deferred `Box` values.

### Refactoring exclusions

*Batch 2025-06.*

CDK refactoring excludes `lambda.Version` and `apigateway.Deployment` resources.

### Scoped removal policies

*Batch 2025-03.*

Core exposes `RemovalPolicies.of(scope)` as the scope-oriented entry point for applying removal policies.

### Suppressible informational annotations

*Batch 2025-07.*

Core annotations provide `addInfoV2` for informational messages that consumers can suppress.

## Core type and packaging contracts

### Bundling aws-cdk-lib

*Batch 2026-04.*

Packages can use `aws-cdk-lib` as a `bundledDependency`; the previous packaging failure was fixed.

### Narrower encrypted-resource interface

*Batch 2026-02.*

`IEncryptedResource` now extends `IEnvironmentAware` rather than `IResource`, and `GrantableResources.isEncryptedResource()` now accepts `IEnvironmentAware` rather than `IConstruct`. Code that still needs an `IResource` must use `IEncryptedResource & IResource` or guard with `Resource.isResource()`.

### Narrower resource-reference types

*Batch 2026-01.*

Several exposed values now use reference interfaces: `JobQueue.computeEnvironments[].computeEnvironment` uses `IComputeEnvironmentRef`, `BackupPlanRule.props.backupVault` uses `IBackupVaultRef`, `EventDestination.bus` uses `IEventBusRef`, and log-group results use `ILogGroupRef`; `ApiDestination.fromApiDestinationAttributes()` now returns `IApiDestination`. Code that needs members from the richer L2 interfaces must type-test or cast these values.

## Cross-stack references and environments

### Cross-region stack outputs

*Batch 2026-05.*

Core adds `Fn::GetStackOutput` for cross-region references. Cross-region references also avoid the earlier “exports cannot be updated” failure.

### Environment-aware constructs

*Batch 2025-11.*

Core exposes `IEnvironmentAware` for retrieving a construct's environment.

### Weak cross-stack references

*Batch 2026-05.*

Core supports weak cross-stack references both within the same environment and across environments.

### Weak-reference guidance and list attributes

*Batch 2026-06.*

Core now recommends weak references when reference strength has not been chosen. Weak cross-stack references also work with list-valued attributes instead of failing.

## Custom resources

### Async custom-resource logging default

*Batch 2025-07.*

Logging in the asynchronous custom-resource provider framework defaults to off.

### AwsCustomResource external IDs

*Batch 2025-11.*

`AwsCustomResource` supports an external ID.

### Custom-resource service timeout

*Batch 2025-01.*

Custom resources accept a `serviceTimeout` property, allowing the service-side operation timeout to be configured independently.

### Node.js 22 custom-resource default

*Batch 2025-05.*

Custom resources now default to the Node.js 22 runtime in commercial, China, and government regions.

### Regional custom-resource runtime default

*Batch 2025-02.*

The default custom-resource Node.js runtime in China and government regions is Node.js 20.

## Synthesis and cloud assemblies

### Additional context cache keys

*Batch 2025-06.*

CDK context values can include an additional cache key, allowing otherwise identical lookups to occupy distinct cache entries.

### Feature flags in cloud assemblies

*Batch 2025-07.*

Cloud assemblies report feature-flag configuration and feature-flag information for downstream consumers.

### Git source metadata in templates

*Batch 2026-07.*

Synthesized CloudFormation templates can now carry Git source metadata.

### Nested-stack indentation suppression

*Batch 2026-02.*

Nested stacks can suppress indentation in their synthesized templates.

## Validation and diagnostics

### Additional typed validation failures

*Batch 2025-06.*

Custom Resources, IAM, Region Info, Secrets Manager, Service Catalog, and additional core paths now throw typed validation errors; Region Info also exports its error types from `region-info/lib/fact`.

### App validation plugins

*Batch 2026-04.*

`Validations` is now the entry point for adding validation plugins to CDK apps and supports `addWarning`, `addError`, and `acknowledge`. Policy-validation interfaces have also graduated from `policyValidationBeta1` to `policyValidation`.

### Boolean context validation

*Batch 2026-06.*

Core validation correctly handles the string `"false"` when a boolean context value is expected.

### CloudFormation include with AWS::NoValue

*Batch 2025-09.*

CloudFormation include accepts `AWS::NoValue` for non-string properties without a type-validation error.

### Comprehensive built-in template validation

*Batch 2026-07.*

Core now validates templates against a comprehensive default rule set. Set `CDK_VALIDATION=false` to disable built-in template validation for a CDK invocation.

```sh
CDK_VALIDATION=false cdk synth
```

### Error codes

*Batch 2026-03.*

CDK errors and error annotations now carry error codes, allowing callers to classify failures without parsing messages.

### Expanded typed validation failures

*Batch 2025-04.*

Kinesis, Cognito Identity Pools, FSx, and SES now throw typed validation errors instead of untyped errors.

### External construct-error traces

*Batch 2026-06.*

`ConstructError` can carry external traces, and available external stack traces are appended to cloud-assembly metadata for better source diagnostics.

### Self-contained validation reports

*Batch 2026-07.*

Validation reports are now self-contained, and Cloud Assembly relative-path handling has been corrected.

### Typed validation failures

*Batch 2025-01.*

Construct validation in API Gateway v2 and its authorizers, ELBv2, Lambda, RDS, Route 53, S3, SNS, SQS, SSM, and Synthetics now throws `ValidationError` instead of untyped errors. The CLI also emits typed errors, allowing callers to distinguish error classes without parsing messages.

### Validation report controls

*Batch 2026-05.*

Validation reports are now always written to the cloud assembly and include construct annotations. The `failSynthOnValidationErrors` context key can suppress validation-error console output and the exit code, and plugin violations can also be suppressed.

### Validation report schema

*Batch 2026-06.*

`validation-report.json` uses a new schema and includes suppressed violations, so report consumers must account for violations that did not fail synthesis.

### Validation-plugin context and outputs

*Batch 2026-06.*

`IPolicyValidationContext` now exposes `scope`, and validation plugins can create files in the cloud assembly.
