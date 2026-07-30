# Networking Apis And Edge

API Gateway, CloudFront, Route 53, Elastic Load Balancing, EC2, VPC, and Auto Scaling. Items are organized by topic; parenthetical identifiers preserve exact extraction-batch attribution.

## Additional API Gateway configuration (`2025-11`)

`SpecRestApi` accepts `binaryMediaTypes`, and API Gateway v2 `WebSocketStage` accepts `accessLogSettings`.

## Alias-record TTL diagnostics (`2025-06`)

Route 53 `RecordSet` warns when a TTL is supplied together with an alias target.

## API Gateway response streaming (`2025-11`)

API Gateway constructs support response streaming with a configurable response transfer mode.

## API Gateway REST base properties (`2025-03`)

`endpointConfiguration` is now defined on `RestApiBaseProps`.

## API Gateway TLS 1.3 domain policies (`2026-03`)

API Gateway domain names support TLS 1.3 security policies.

## API Gateway v2 stage variables (`2025-07`)

HTTP and WebSocket API stages support stage variables.

## Application Load Balancer JWT verification (`2026-04`)

Elastic Load Balancing v2 constructs support JWT verification for Application Load Balancers.

## ARecord delete-existing deprecation (`2025-08`)

The delete-existing field on `ARecord` is deprecated.

## Auto Scaling availability-zone distribution (`2025-01`)

`AutoScalingGroup` accepts `availabilityZoneDistribution` to control capacity distribution across Availability Zones.

## Auto Scaling lifecycle controls (`2026-03`)

`AutoScalingGroup` accepts `deletionProtection` and `instanceLifecyclePolicy`.

## BYOIP IPv6 for VpcV2 (`2025-01`)

`VpcV2` can use bring-your-own-IP IPv6 addressing.

## Client VPN automatic reconnect (`2025-10`)

EC2 Client VPN endpoint constructs support automatic VPN-session reconnect.

## Cloud WAN core-network routes (`2025-11`)

EC2 constructs support routes for Cloud WAN core networks.

## CloudFront Functions JavaScript 2.0 default (`2026-03`)

Under its feature flag, CloudFront Functions now default to the JavaScript 2.0 runtime.

## CloudFront gRPC (`2025-02`)

CloudFront distributions can be configured for gRPC traffic.

## CloudFront host-header-only origin policy (`2026-07`)

CloudFront exposes the `Managed-HostHeaderOnly` managed origin request policy.

## CloudFront HTTP-origin controls (`2025-09`)

HTTP origins can select an IP-address type and configure a response-completion timeout.

## CloudFront origin-group selection (`2025-02`)

L2 CloudFront distributions and origin groups support origin-group selection criteria.

## CloudFront VPC origins (`2025-02`)

CloudFront distributions can use origins hosted inside a VPC.

## Cross-region VPC endpoints (`2025-08`)

`AWS::EC2::VPCEndpoint` exposes the `ServiceRegion` property.

## Dual-stack API Gateway v2 APIs (`2025-04`)

API Gateway v2 constructs support dual-stack HTTP and WebSocket APIs.

## Dual-stack API Gateway v2 domains (`2025-05`)

API Gateway v2 domain names support dual-stack addressing.

## Dual-stack REST APIs (`2025-05`)

API Gateway REST API constructs support dual-stack addressing.

## EC2 C8A instances (`2026-05`)

The EC2 instance-type catalog includes C8A.

## EC2 C8GN instances (`2025-07`)

The EC2 instance-class catalog includes C8GN.

## EC2 deployment actions (`2025-06`)

CodePipeline action constructs support native Amazon EC2 deployments.

## EC2 Fleet replacement constraints (`2025-12`)

`AWS::EC2::EC2Fleet` now treats `DefaultTargetCapacityType` and `TargetCapacityUnitType` as immutable, so changing either property replaces the fleet rather than updating it in place.

## EC2 instance metadata options (`2025-12`)

EC2 instance constructs expose their `MetadataOptions` configuration for callers that need to inspect the instance metadata settings.

## Exportable public certificates (`2025-09`)

ACM constructs support exportable public certificates.

## Feature-flagged HTTPS redirect distribution (`2025-12`)

Under its feature flag, Route 53 patterns' `HttpsRedirect` uses CloudFront `Distribution` as its default distribution implementation.

## Feature-flagged NLB security groups (`2025-11`)

Under its feature flag, Elastic Load Balancing v2 creates Network Load Balancer security-group settings by default.

## HTTP API stage access logging (`2025-04`)

API Gateway v2 `HttpStage` supports access logging.

## Load-balancer mTLS CA-name advertisement (`2025-02`)

Elastic Load Balancing v2 supports `AdvertiseTrustStoreCaNames` for mutual TLS.

## Minimum load-balancer capacity (`2025-02`)

Elastic Load Balancing v2 constructs support minimum Load Balancer Capacity Unit reservations.

## Multiple Auto Scaling health checks (`2025-03`)

The new `HealthChecks` API supports multiple health-check types, including EBS and `VPC_LATTICE`.

## Network Load Balancer subnet mappings (`2025-04`)

Elastic Load Balancing v2 constructs support subnet mappings for Network Load Balancers.

## Prefix lists as connection peers (`2025-06`)

An EC2 `PrefixList` now implements `IPeer`, so it can be passed directly to connection and security-group rule APIs.

## Private API resource policies (`2025-02`)

API Gateway constructs support resource-policy configuration for private APIs.

## Restricted Route 53 delegation (`2025-11`)

Route 53 `grantDelegation` can restrict the delegated zone names.

## Route 53 accelerated recovery (`2026-04`)

Public hosted-zone constructs support accelerated recovery.

## Route 53 failover records (`2025-12`)

Route 53 record-set constructs support failover routing policies.

## SpecRestApi deployment mode (`2025-04`)

`SpecRestApi` accepts a `mode` property.

## SVCB and HTTPS DNS records (`2025-09`)

Route 53 provides resource-record classes for SVCB and HTTPS records.

## Target-group health attributes (`2025-09`)

Elastic Load Balancing v2 target groups support health attributes.

## Token-safe Elastic Beanstalk aliases (`2025-05`)

Elastic Beanstalk Route 53 targets accept `hostedZoneId` for tokenized endpoints, defaulting it from the stack region or `endpointUrl`.

## Versioned CloudFront origin reads (`2025-02`)

CloudFront origins support a versioned-read access level.

## WebSocket API usage plans and API keys (`2025-08`)

API Gateway v2 L2 constructs now support usage plans and API keys for `WebSocketApi`.

## WebSocket schema-validation opt-out (`2025-09`)

WebSocket APIs accept `disableSchemaValidation` to bypass schema validation.
