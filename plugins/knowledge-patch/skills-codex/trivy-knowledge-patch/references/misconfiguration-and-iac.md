# Misconfiguration and Infrastructure as Code

## Check metadata, findings, and ignores

- Check metadata accepts an `examples` field (since 0.59.0).
- Inline-comment ignores apply to Dockerfile and Helm misconfiguration scans
  (since 0.59.0).
- Terraform findings render their causes in reports (since 0.60.0).
- Misconfiguration content can declare `Minimum Trivy Version` (since 0.63.0).
- The schema accepts an `audit` configuration attribute (since 0.66.0).
- Rego can ignore findings by type, and scanning accepts a configurable Rego
  error limit. Manifest diagnostics include map keys (since 0.68.0).
- Boolean check-metadata fields are parsed as booleans. An ignore-marker value
  must be known and non-null before it suppresses a result (since 0.68.0).
- Provider mappings use `ID` instead of `AVDID`; migrate custom structures and
  consumers of the previous field (since 0.69.0).
- `.trivyignore` applies check aliases when filtering misconfiguration results
  (since 0.70.0).
- Misconfiguration ignore identifiers match case-insensitively (since 0.71.0).

## Terraform and OpenTofu evaluation

- Terraform parser options apply to submodules as well as the root module
  (since 0.60.0).
- `ephemeral` block types are recognized (since 0.61.0).
- Documents in subdirectories are loaded instead of skipped (since 0.61.0).
- Missing variables evaluate as unknown. Module instances and repeated block
  instances use the correct evaluation context, and HCL object expressions
  return references (since 0.62.0).
- Raw Terraform data is exported to Rego policies, and the parser accepts an
  explicit current working directory (since 0.63.0).
- Dynamic blocks check that `for_each` is known before expansion (since
  0.63.0).
- OpenTofu-specific extensions are discovered as scan inputs (since 0.64.0).
- Terraform policy templates support partial evaluation (since 0.64.0).
- Map-valued `for_each` creates one resource per key (since 0.65.0).
- Terraform plan scans use remote modules cached in `.terraform`; cached remote
  submodules retain their original paths (since 0.66.0).
- OpenTofu files participate in module detection (since 0.67.0).
- Terraform `action` blocks are recognized, and plan configuration can
  partially restore schema information (since 0.69.0).
- Nested values in Terraform-plan lists render correctly; resources with no
  `after` changes are skipped (since 0.71.0).
- Terraform filesystem functions prevent path traversal (since 0.71.0).

## Rego integration and generic parsing

- The IaC scanner accepts an injected Rego scanner (since 0.62.0).
- Null nodes are removed from JSON manifests before misconfiguration evaluation
  (since 0.62.0).
- JSONC inputs allow comments and trailing commas (since 0.63.0).

## AWS and CloudFormation

- ECS definitions recognize enhanced Container Insights (since 0.60.0).
- Terraform adapters cover `aws_default_security_group`,
  `aws_opensearch_domain`, and `aws_ami`. CloudFormation adapters cover
  `AWS::DynamoDB::Table`, `AWS::EC2::VPC`, and defaults for
  `AWS::EKS::Cluster.ResourcesVpcConfig` (since 0.61.0).
- AWS managed policies are converted into evaluable policy documents (since
  0.62.0).
- `Fn::FindInMap` supports default values and list-valued results (since
  0.67.0).
- CloudFormation evaluation supports `Fn::ForEach` (since 0.69.0).
- `AWS::EC2::Instance` propagates `MetadataOptions` to checks (since 0.71.0).
- Check `AVD-AWS-0010` recognizes CloudFront standard logging v2 (since
  0.72.0).

## Azure, GCP, and GitHub resources

- Azure `CreateUiDefinition` documents are skipped (since 0.61.0).
- Terraform `google_container_cluster` evaluation supports
  `auto_provisioning_defaults` (since 0.62.0).
- GCP subnetworks expose private IP Google access, and storage buckets expose
  logging and versioning (since 0.65.0).
- GCP bucket logging uses `log_bucket`, not `target_bucket` (since 0.66.0).
- Azure schema support includes agent pools, role assignments,
  `https_traffic_only_enabled` for storage accounts, and expanded App Service,
  Compute, Container, Network, Storage, and Security Center resources (since
  0.68.0).
- Azure ARM resources expressed as objects, `azurerm_*_web_app` resources, and
  expanded Azure Database schemas are recognized (since 0.69.0).
- Azure Resource Manager Kubernetes clusters are adapted, resources resolve
  through `resource_id`, and Terraform supports
  `azurerm_network_interface_security_group_association` (since 0.70.0).
- The `github_repository_vulnerability_alerts` resource is supported (since
  0.72.0).

## Helm and Kubernetes content

- Helm detection identifies the chart file by exact filename rather than loose
  name matching (since 0.61.0).
- Kubernetes scans collect components from namespaced resources (since
  0.63.0).
- Ignore paths include the chart path when a chart is in a subdirectory (since
  0.66.0).
- The Helm deployment accepts `sslCertDir` for an SSL certificate directory
  (since 0.69.0).
- `.yml` files participate in Helm-chart detection (since 0.69.0).

## Dockerfile and image evaluation

- Image history omits check `AVD-DS-0007` (since 0.60.0).
- Buildah and legacy Docker-builder `CreatedBy` values are normalized (since
  0.64.0).
- `.Config.User` takes precedence over `USER` history entries when calculating
  the effective image user (since 0.64.0).
- Build-metadata suffixes are removed from history, and legacy `ENV` values are
  quoted to retain spaces (since 0.67.0).
- Unsupported experimental Dockerfile flags are tolerated, and health-check
  start period maps to `--start-period` (since 0.68.0).
- `RUN` instructions from non-BuildKit images are reconstructed correctly
  (since 0.71.0).
- Docker configuration consumers must migrate to the breaking `dockers_v2`
  representation (since 0.72.0).

## Additional content

- Initial Ansible misconfiguration scanning is supported (since 0.69.0).

