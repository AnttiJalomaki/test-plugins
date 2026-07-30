# Identity, security, and AI services

IAM, KMS, Cognito, Secrets Manager, Bedrock, AgentCore, SageMaker, and security-sensitive grants.

## Bedrock and AgentCore

### Added Bedrock models

*Batch 2025-04.*

The Bedrock model catalog adds Amazon Nova Sonic 1.0 and Nova Reel 1.1.

### Added Bedrock models

*Batch 2026-03.*

The Bedrock foundation-model catalog adds MiniMax and GLM identifiers.

### AgentCore Cognito authorizers

*Batch 2025-11.*

`RuntimeAuthorizerConfiguration.usingCognito()` now accepts `IUserPool` and `IUserPoolClient` constructs instead of string identifiers and supports multiple clients.

### AgentCore gateway IAM credential targets

*Batch 2026-07.*

Bedrock AgentCore gateway-target IAM credential providers can specify the target service and region.

### AgentCore interface endpoints

*Batch 2025-10.*

`InterfaceVpcEndpointAwsService` includes `BEDROCK_AGENTCORE` and `BEDROCK_AGENTCORE_GATEWAY`.

### AgentCore L2 constructs

*Batch 2025-10.*

AgentCore now provides L2 constructs for runtimes and first-party tools.

### AgentCore memory

*Batch 2025-11.*

AgentCore provides an L2 construct for memory.

### Bedrock agent prompt management

*Batch 2025-07.*

Bedrock agent constructs support prompt management.

### Bedrock DeepSeek R1

*Batch 2025-03.*

Bedrock constructs support the DeepSeek R1 model.

### Deprecated Bedrock models

*Batch 2025-01.*

Bedrock model entries for Claude 2, Claude 2.1, and Claude Instant are deprecated.

### Inference profiles

*Batch 2025-08.*

Inference-profile constructs support inference and cross-region inference profiles.

### Ray2 model support

*Batch 2025-02.*

The Bedrock model catalog includes the Ray2 visual model.

### Required AgentCore reference getters

*Batch 2026-02.*

AgentCore interface implementors must now provide `gatewayRef`, `gatewayTargetRef`, `memoryRef`, `runtimeRef`, `runtimeEndpointRef`, `browserCustomRef`, or `codeInterpreterCustomRef` on the corresponding `IGateway`, `IGatewayTarget`, `IMemory`, `IBedrockAgentRuntime`, `IRuntimeEndpoint`, `IBrowserCustom`, and `ICodeInterpreterCustom` interfaces.

### Stable Bedrock AgentCore constructs

*Batch 2026-05.*

Bedrock AgentCore has graduated to stable.

## Cognito

### Cognito choice-based authentication

*Batch 2025-02.*

Cognito constructs support choice-based authentication, including passwordless and passkey sign-in.

### Cognito client analytics

*Batch 2025-02.*

Cognito user-pool clients accept analytics configuration.

### Cognito managed login

*Batch 2025-01.*

Cognito constructs gained support for managed login.

### Cognito pre-token trigger event v3

*Batch 2025-04.*

Cognito constructs support version 3.0 of the pre-token-generation trigger event.

### Cognito refresh-token rotation

*Batch 2025-09.*

Cognito constructs support refresh-token rotation.

### Stable Cognito identity pools

*Batch 2025-03.*

Cognito Identity Pool constructs graduated from experimental to stable.

## IAM and grants

### Grants for imported KMS aliases

*Batch 2025-06.*

Under its feature flag, aliases imported with `Alias.fromAliasName()` support grant methods.

### Native OIDC providers

*Batch 2025-06.*

IAM provides `OidcProviderNative`, which uses the native CloudFormation OIDC-provider resource rather than a custom resource.

### Optional KMS account-identity trust

*Batch 2026-02.*

`trustAccountIdentities` is now optional in `KeyGrants`.

## KMS and secrets

### KMS alias behavior

*Batch 2025-09.*

`Alias` references resolve to the alias instead of its underlying key. Aliases imported with `Alias.fromAliasName()` expose `aliasTargetKey`.

### Literal secret dynamic-reference keys

*Batch 2025-11.*

`SecretValue` and Secrets Manager `Secret` provide methods for obtaining a literal dynamic-reference key that CloudFormation does not resolve.

### Python custom-resource runtimes

*Batch 2025-09.*

Python custom resources use Python 3.13. Secrets Manager's `SecretRotationApplication` no longer creates an EOL Python 3.9 function.

## SageMaker

### SageMaker serverless inference

*Batch 2025-11.*

SageMaker constructs support serverless inference endpoints.
