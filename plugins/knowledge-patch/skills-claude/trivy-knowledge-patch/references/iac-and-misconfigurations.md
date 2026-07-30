# IaC and Misconfigurations

## Input discovery and parser behavior

- Dockerfile and Helm scans honor inline-comment ignores (since 0.59.0).
- Terraform parser options apply to submodules as well as the root
  (since 0.60.0).
- Filesystem post-analyzers inspect static paths, and `--file-patterns` reaches
  every post-analyzer (since 0.61.0).
- The schema recognizes `ephemeral` blocks, and documents below nested
  subdirectories are loaded (since 0.61.0).
- Helm chart detection requires the exact chart filename, preventing loose-name
  matches. Azure `CreateUiDefinition` documents are excluded
  (since 0.61.0).
- JSON manifest parsing filters null nodes (since 0.62.0).
- JSONC comments and trailing commas are accepted (since 0.63.0).
- Terraform parsing can set its current working directory (since 0.63.0).
- OpenTofu-specific extensions are discovered (since 0.64.0) and participate in
  module detection (since 0.67.0).
- Helm chart detection includes `.yml` files (since 0.69.0).
- Initial Ansible misconfiguration scanning is available (since 0.69.0).

## Terraform and OpenTofu evaluation

- An IaC scanner can receive a caller-provided Rego scanner (since 0.62.0).
- Missing variables evaluate as unknown values. Module instances and repeated
  block instances use the correct evaluation context, and HCL object expressions
  return their references (since 0.62.0).
- Raw Terraform data is exposed to Rego policies (since 0.63.0).
- Before expanding a dynamic block, evaluation checks whether `for_each` is
  known (since 0.63.0).
- Policy templates support partial evaluation (since 0.64.0).
- A `for_each` map expands into one resource per key (since 0.65.0).
- Terraform plan scans use remote modules cached in `.terraform`; cached remote
  submodules retain their original paths (since 0.66.0).
- Terraform `action` blocks are recognized, and plan configuration can partially
  restore schema information (since 0.69.0).
- Nested list values in plans render correctly, and resources without `after`
  changes are skipped (since 0.71.0).
- Terraform filesystem functions prevent path traversal during evaluation
  (since 0.71.0).

Unknown values must remain unknown until the evaluator can resolve them. Do not
coerce them to false, empty collections, or a guessed iteration count.

## Kubernetes and Helm

- Kubernetes scans support controllers and emit complete output for
  `--report all` (since 0.61.0).
- Artifact versions compare correctly, and scanning no longer reads the
  `last-applied-configuration` annotation as the artifact source
  (since 0.62.0).
- Summary reports omit passed misconfigurations (since 0.62.0).
- Namespaced Kubernetes resources contribute components (since 0.63.0).
- Chart ignore rules account for a chart located in a subdirectory
  (since 0.66.0).
- The Helm deployment supports `sslCertDir` for an SSL certificate directory
  (since 0.69.0).

## Docker and image configuration

- Inline ignores are supported in Dockerfiles (since 0.59.0).
- Image-history scanning no longer runs check `AVD-DS-0007`
  (since 0.60.0).
- Buildah and legacy Docker `CreatedBy` history values are normalized
  (since 0.64.0).
- `.Config.User` always outranks `USER` history entries when determining the
  effective image user (since 0.64.0).
- Image history strips build-metadata suffixes and quotes legacy `ENV` values so
  spaces remain intact (since 0.67.0).
- Unsupported experimental Dockerfile flags do not break parsing; health-check
  start period maps to `--start-period` (since 0.68.0).
- Non-BuildKit images reconstruct `RUN` instructions correctly
  (since 0.71.0).

## Rego, check metadata, and ignores

- Check metadata can include `examples` (since 0.59.0) and
  `Minimum Trivy Version` (since 0.63.0).
- Rego findings can be ignored by type, and callers can configure a Rego error
  limit (since 0.68.0).
- Boolean check-metadata values are parsed as booleans. An ignore marker applies
  only when its value is known and non-null (since 0.68.0).
- Provider mappings use `ID` rather than `AVDID`; custom mappings and consumers
  must migrate (breaking since 0.69.0).
- `.trivyignore` filtering applies check aliases (since 0.70.0).
- Misconfiguration ignore identifiers compare case-insensitively
  (since 0.71.0).
- The configuration supports an `audit` attribute (since 0.66.0).

## AWS and CloudFormation

- ECS definitions recognize enhanced Container Insights (since 0.60.0).
- Resource adapters cover Terraform `aws_default_security_group`,
  `aws_opensearch_domain`, and `aws_ami`; CloudFormation adapters cover
  `AWS::DynamoDB::Table`, `AWS::EC2::VPC`, and default values for
  `AWS::EKS::Cluster.ResourcesVpcConfig` (since 0.61.0).
- AWS managed policies are converted into documents for policy evaluation
  (since 0.62.0).
- `Fn::FindInMap` supports default values and list-valued results
  (since 0.67.0).
- `Fn::ForEach` is evaluated (since 0.69.0).
- `AWS::EC2::Instance` propagates `MetadataOptions` to checks
  (since 0.71.0).
- Check `AVD-AWS-0010` recognizes CloudFront standard logging v2
  (since 0.72.0).

## Azure

- Schema support includes agent pools, role assignments, storage-account
  `https_traffic_only_enabled`, and expanded App Service, Compute, Container,
  Network, Storage, and Security Center resources (since 0.68.0).
- ARM resources expressed as objects, `azurerm_*_web_app`, expanded Azure
  Database resources, and Terraform `action` blocks are recognized
  (since 0.69.0).
- Azure Resource Manager Kubernetes clusters are adapted, references through
  `resource_id` are resolved, and
  `azurerm_network_interface_security_group_association` is supported
  (since 0.70.0).

## GCP and GitHub

- Terraform `google_container_cluster` supports `auto_provisioning_defaults`
  (since 0.62.0).
- Subnetwork private-IP Google access and storage-bucket logging/versioning are
  exposed to checks (since 0.65.0).
- GCP bucket logging uses `log_bucket`, not `target_bucket` (since 0.66.0).
- `github_repository_vulnerability_alerts` is supported (since 0.72.0).
