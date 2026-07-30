# Core Toolkit And Validation

Core, toolkit, synthesis, assets, validation, custom resources, and application behavior. Items are organized by topic; parenthetical identifiers preserve exact extraction-batch attribution.

## Additional context cache keys (`2025-06`)

CDK context values can include an additional cache key, allowing otherwise identical lookups to occupy distinct cache entries.

## Additional typed validation failures (`2025-06`)

Custom Resources, IAM, Region Info, Secrets Manager, Service Catalog, and additional core paths now throw typed validation errors; Region Info also exports its error types from `region-info/lib/fact`.

## App validation plugins (`2026-04`)

`Validations` is now the entry point for adding validation plugins to CDK apps and supports `addWarning`, `addError`, and `acknowledge`. Policy-validation interfaces have also graduated from `policyValidationBeta1` to `policyValidation`.

## Asset-bundling platform selection (`2025-05`)

Core asset bundling now honors the configured `platform` instead of ignoring it.

## Async custom-resource logging default (`2025-07`)

Logging in the asynchronous custom-resource provider framework defaults to off.

## Auto-delete mixins (`2026-03`)

ECR adds `RepositoryAutoDeleteImages`, and S3's `BucketAutoDeleteObjects` mixin graduates into `aws-cdk-lib`.

## AwsCustomResource external IDs (`2025-11`)

`AwsCustomResource` supports an external ID.

## Boolean context validation (`2026-06`)

Core validation correctly handles the string `"false"` when a boolean context value is expected.

## Bundling aws-cdk-lib (`2026-04`)

Packages can use `aws-cdk-lib` as a `bundledDependency`; the previous packaging failure was fixed.

## CDK Mixins (`2026-03`)

CDK introduces Mixins as an extension mechanism, graduates `@aws-cdk/cfn-property-mixins` to stable, and provides helpers for converting between Aspects and Mixins. `PropertyMergeStrategy` can merge arbitrary CloudFormation property objects, and S3 and ECS service mixins now ship in `aws-cdk-lib`.

## CloudFormation include with AWS::NoValue (`2025-09`)

CloudFormation include accepts `AWS::NoValue` for non-string properties without a type-validation error.

## Comprehensive built-in template validation (`2026-07`)

Core now validates templates against a comprehensive default rule set. Set `CDK_VALIDATION=false` to disable built-in template validation for a CDK invocation.

```sh
CDK_VALIDATION=false cdk synth
```

## Configurable cdk-assets version (`2025-07`)

CDK Pipelines can configure the `cdk-assets` version.

## Cross-region stack outputs (`2026-05`)

Core adds `Fn::GetStackOutput` for cross-region references. Cross-region references also avoid the earlier “exports cannot be updated” failure.

## Custom bootstrap qualifiers in staging roles (`2025-08`)

The app-staging synthesizer propagates a custom bootstrap qualifier into the deployment-role name.

## Custom-resource service timeout (`2025-01`)

Custom resources accept a `serviceTimeout` property, allowing the service-side operation timeout to be configured independently.

## Deferred values with source traces (`2026-04`)

Core provides a `Box` API for deferred values that preserves accurate stack traces.

## Environment-aware constructs (`2025-11`)

Core exposes `IEnvironmentAware` for retrieving a construct's environment.

## Error codes (`2026-03`)

CDK errors and error annotations now carry error codes, allowing callers to classify failures without parsing messages.

## Expanded typed validation failures (`2025-04`)

Kinesis, Cognito Identity Pools, FSx, and SES now throw typed validation errors instead of untyped errors.

## External construct-error traces (`2026-06`)

`ConstructError` can carry external traces, and available external stack traces are appended to cloud-assembly metadata for better source diagnostics.

## Feature flags in cloud assemblies (`2025-07`)

Cloud assemblies report feature-flag configuration and feature-flag information for downstream consumers.

## Git source metadata in templates (`2026-07`)

Synthesized CloudFormation templates can now carry Git source metadata.

## Gitignore negation in subdirectories (`2025-09`)

Negated gitignore patterns inside subdirectories now re-include matching files.

## Go app-staging synthesizer (`2025-05`)

The experimental `app-staging-synthesizer-alpha` package is now published for Go.

## Nested-stack indentation suppression (`2026-02`)

Nested stacks can suppress indentation in their synthesized templates.

## New platform versions (`2025-11`)

The EKS catalog includes Kubernetes 1.34, and the Lambda runtime catalog includes Node.js 24.x, Java 25, and Python 3.14.

## Node.js 22 custom-resource default (`2025-05`)

Custom resources now default to the Node.js 22 runtime in commercial, China, and government regions.

## Opt-in JSON escaping (`2025-04`)

`Source.jsonData()` no longer escapes JSON automatically. Pass `{ escape: true }` as its third argument when special characters require the former behavior: `Source.jsonData("config.json", data, { escape: true })`.

## Property injection across L2 constructs (`2025-05`)

Property injectors can target all L2 constructors.

## Property merge strategies (`2026-05`)

`PropertyMergeStrategy` now supports array merge strategies, and the built-in strategies are compatible with deferred `Box` values.

## Public CLI plugin contract (`2025-01`)

A public CLI-plugin contract now defines the supported boundary between the CLI and plugins, and imports of internal CLI libraries are disallowed. Credential plugins may return `null` for expiration and may initially return empty credentials.

## Python custom-resource runtimes (`2025-09`)

Python custom resources use Python 3.13. Secrets Manager's `SecretRotationApplication` no longer creates an EOL Python 3.9 function.

## Refactoring exclusions (`2025-06`)

CDK refactoring excludes `lambda.Version` and `apigateway.Deployment` resources.

## Regional custom-resource runtime default (`2025-02`)

The default custom-resource Node.js runtime in China and government regions is Node.js 20.

## Scoped removal policies (`2025-03`)

Core exposes `RemovalPolicies.of(scope)` as the scope-oriented entry point for applying removal policies.

## Self-contained validation reports (`2026-07`)

Validation reports are now self-contained, and Cloud Assembly relative-path handling has been corrected.

## Simplified resource import (`2025-01`)

The CLI supports CloudFormation simplified resource import, reducing the setup required for bringing existing resources under stack management.

## Suppressible informational annotations (`2025-07`)

Core annotations provide `addInfoV2` for informational messages that consumers can suppress.

## Toolkit version environment contract (`2025-03`)

The cloud-assembly API now declares `CDK_TOOLKIT_VERSION` as a supported environment variable.

## Typed validation failures (`2025-01`)

Construct validation in API Gateway v2 and its authorizers, ELBv2, Lambda, RDS, Route 53, S3, SNS, SQS, SSM, and Synthetics now throws `ValidationError` instead of untyped errors. The CLI also emits typed errors, allowing callers to distinguish error classes without parsing messages.

## Validation report controls (`2026-05`)

Validation reports are now always written to the cloud assembly and include construct annotations. The `failSynthOnValidationErrors` context key can suppress validation-error console output and the exit code, and plugin violations can also be suppressed.

## Validation report schema (`2026-06`)

`validation-report.json` uses a new schema and includes suppressed violations, so report consumers must account for violations that did not fail synthesis.

## Validation-plugin context and outputs (`2026-06`)

`IPolicyValidationContext` now exposes `scope`, and validation plugins can create files in the cloud assembly.

## Weak cross-stack references (`2026-05`)

Core supports weak cross-stack references both within the same environment and across environments.

## Weak-reference guidance and list attributes (`2026-06`)

Core now recommends weak references when reference strength has not been chosen. Weak cross-stack references also work with list-valued attributes instead of failing.
