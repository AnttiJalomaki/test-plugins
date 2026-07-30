# Identity Security And Ai

IAM, KMS, Cognito, secrets, certificates, Bedrock, AgentCore, SageMaker, and SES. Items are organized by topic; parenthetical identifiers preserve exact extraction-batch attribution.

## ACM interface endpoints (`2025-12`)

EC2 constructs provide interface VPC endpoint services for ACM and ACM Private CA.

## Added Bedrock models (`2025-04`)

The Bedrock model catalog adds Amazon Nova Sonic 1.0 and Nova Reel 1.1.

## Added Bedrock models (`2026-03`)

The Bedrock foundation-model catalog adds MiniMax and GLM identifiers.

## AgentCore Cognito authorizers (`2025-11`)

`RuntimeAuthorizerConfiguration.usingCognito()` now accepts `IUserPool` and `IUserPoolClient` constructs instead of string identifiers and supports multiple clients.

## AgentCore gateway IAM credential targets (`2026-07`)

Bedrock AgentCore gateway-target IAM credential providers can specify the target service and region.

## AgentCore interface endpoints (`2025-10`)

`InterfaceVpcEndpointAwsService` includes `BEDROCK_AGENTCORE` and `BEDROCK_AGENTCORE_GATEWAY`.

## AgentCore L2 constructs (`2025-10`)

AgentCore now provides L2 constructs for runtimes and first-party tools.

## AgentCore memory (`2025-11`)

AgentCore provides an L2 construct for memory.

## Bedrock agent prompt management (`2025-07`)

Bedrock agent constructs support prompt management.

## Bedrock DeepSeek R1 (`2025-03`)

Bedrock constructs support the DeepSeek R1 model.

## Bedrock model-customization jobs (`2025-05`)

Step Functions task integrations support Bedrock `CreateModelCustomizationJob`.

## Bootstrap trust removal (`2025-01`)

`cdk bootstrap` accepts `--untrust`, providing a direct way to retract bootstrap trust.

## Cognito choice-based authentication (`2025-02`)

Cognito constructs support choice-based authentication, including passwordless and passkey sign-in.

## Cognito client analytics (`2025-02`)

Cognito user-pool clients accept analytics configuration.

## Cognito managed login (`2025-01`)

Cognito constructs gained support for managed login.

## Cognito pre-token trigger event v3 (`2025-04`)

Cognito constructs support version 3.0 of the pre-token-generation trigger event.

## Cognito refresh-token rotation (`2025-09`)

Cognito constructs support refresh-token rotation.

## Deprecated Bedrock models (`2025-01`)

Bedrock model entries for Claude 2, Claude 2.1, and Claude Instant are deprecated.

## Grants for imported KMS aliases (`2025-06`)

Under its feature flag, aliases imported with `Alias.fromAliasName()` support grant methods.

## Inference profiles (`2025-08`)

Inference-profile constructs support inference and cross-region inference profiles.

## KMS alias behavior (`2025-09`)

`Alias` references resolve to the alias instead of its underlying key. Aliases imported with `Alias.fromAliasName()` expose `aliasTargetKey`.

## KMS-encrypted AppConfig hosted configurations (`2025-06`)

AppConfig hosted configurations support encryption with a customer-managed key.

## Literal secret dynamic-reference keys (`2025-11`)

`SecretValue` and Secrets Manager `Secret` provide methods for obtaining a literal dynamic-reference key that CloudFormation does not resolve.

## Narrower pipeline-role trust (`2025-03`)

Under `@aws-cdk/pipelines:reduceStageRoleTrustScope`, the trust policy uses the current pipeline role instead of the account root principal.

## Native OIDC providers (`2025-06`)

IAM provides `OidcProviderNative`, which uses the native CloudFormation OIDC-provider resource rather than a custom resource.

## Optional KMS account-identity trust (`2026-02`)

`trustAccountIdentities` is now optional in `KeyGrants`.

## Ray2 model support (`2025-02`)

The Bedrock model catalog includes the Ray2 visual model.

## SageMaker serverless inference (`2025-11`)

SageMaker constructs support serverless inference endpoints.

## Separate grants APIs (`2025-11`)

Grant operations are available through separate classes with public factory methods. November adds `StateMachineGrants`, `TableGrants`, `StreamGrants`, `BucketGrants`, and `HostedZoneGrants`.

## SES configuration-set email validation (`2026-05`)

SES configuration sets support automatic email validation.

## SES custom-tracking HTTPS policy (`2025-05`)

SES constructs support an HTTPS policy for custom tracking domains.

## SES event destinations (`2025-02`)

SES configuration sets support the default event bus and Firehose as event destinations.

## Stable Bedrock AgentCore constructs (`2026-05`)

Bedrock AgentCore has graduated to stable.

## Stable Cognito identity pools (`2025-03`)

Cognito Identity Pool constructs graduated from experimental to stable.

## Synthesized security behavior (`2025-09`)

`BucketNotificationsHandler` scopes IAM permissions to specific bucket ARNs, and ECS patterns keep `openListener` false when given a custom security group. The EKS kubectl provider uses the `AmazonEC2ContainerRegistryPullOnly` managed policy.
