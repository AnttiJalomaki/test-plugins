# SBOM and Reports

## Package and application structure

- Duplicate `dpkg` packages found at different paths in separate image layers
  are consolidated (since 0.59.0).
- Nested packages attach to their application, and an unknown dependency is
  associated with a root package when one exists (since 0.59.0).
- Applications of the same type from different SBOM files remain distinct, and
  OS packages from multiple SBOM inputs are preserved (since 0.59.0 and
  0.60.0).
- When no better application path is available, the SBOM file's path becomes
  the application's `FilePath` (since 0.60.0).
- Image layer data is included in SBOM scan results (since 0.59.0), and image
  scanning saves layer metadata for downstream report consumers
  (since 0.62.0).
- OS packages found inside and outside an SBOM dependency graph are merged into
  scan results (since 0.65.0).
- Red Hat `BuildInfo` survives SBOM ingestion even without layer information
  (since 0.70.0).

Preserve graph edges, source-file distinctions, application paths, layer data,
and root associations. Flattening on package name or application type loses
meaning required by vulnerability, license, and report consumers.

## CycloneDX ingestion

- External VEX files referenced by a CycloneDX SBOM can be loaded
  (since 0.60.0).
- Tool metadata includes `manufacturer` (since 0.64.0).
- SHA-512 component hashes are understood, and generated CycloneDX licenses use
  the correct field (since 0.65.0).
- Components may carry multiple license types, and components of type `file`
  are supported (since 0.66.0).
- Updating vulnerabilities in a CycloneDX file preserves the input SBOM
  structure (since 0.67.0).
- CycloneDX vulnerability output includes CVSS v4 ratings (since 0.70.0).
- CycloneDX 1.7 is supported (since 0.71.0).

## SPDX ingestion and output

- Licenses outside the SPDX list are emitted through
  `hasExtractedLicensingInfos` (since 0.59.0).
- Plain SPDX text licenses are retained in `otherLicenses` without normalization
  (since 0.61.0).
- SBOMs embedded in SPDX attestations can be consumed (since 0.68.0).
- SPDX license logic distinguishes identifiers, expressions, and `WITH`
  exceptions; literal `unlicensed` is not rewritten to `Unlicense`
  (since 0.68.0).
- Non-library packages use `NOASSERTION` for both `licenseDeclared` and
  `licenseConcluded` (since 0.70.0).
- SPDX serialization supports SHA-512 (since 0.71.0).
- The SPDX marshaler tolerates a document without a root component instead of
  dereferencing a missing value (since 0.72.0).

## Attested and generated SBOMs

- Docker archives retain `RepoTags`, and SBOMs can be read from Sigstore bundles
  or SPDX attestations (since 0.68.0).
- SBOM output exposes `buildInfo` as properties; client/server RPC carries the
  same information in `BlobInfo` (since 0.68.0).
- Embedded-SBOM image scans are deterministic (since 0.69.0).
- The `sbom` command does not accept active `--skip-dir` or `--skip-files`
  behavior (since 0.63.0).

## Report identity and metadata

- Repository scans add Git metadata (since 0.65.0); filesystem cache hits keep
  `RepoMetadata`, and repository URLs are sanitized before reporting
  (since 0.66.0).
- Targets expose `ArtifactID`, incorporating registry and repository. Reports
  expose UUIDv7 `ReportID`, vulnerability fingerprints, and image-reference
  metadata (since 0.68.0).
- Filesystem targets with Git information are reported as repository artifacts
  (since 0.68.0).
- Packages expose the detecting analyzer through `AnalyzedBy` (since 0.69.0).
- JSON reports include the Trivy version (since 0.69.0), and client/server JSON
  also includes the server version (since 0.70.0).
- Client/server transport retains package repository class (since 0.72.0).

## SARIF, JUnit, and templates

- SARIF `shortDescription` and `fullDescription` text remains unescaped rather
  than being HTML-escaped (since 0.60.0).
- The JUnit template includes source locations for misconfiguration findings
  (since 0.63.0).
- SARIF vulnerability results include CVSS vectors (since 0.65.0).
- SARIF uses the correct `ROOTPATH` URI for a Git repository target
  (since 0.70.0).
- The `gitlab.tpl` link array does not contain a trailing comma
  (since 0.71.0).
- Flag validation rejects template files with invalid extensions
  (since 0.70.0).

## Misconfiguration report details

- Terraform findings render their causes in report output (since 0.60.0).
- Kubernetes complete reports honor `--report all` (since 0.61.0), while summary
  reports omit passed misconfigurations (since 0.62.0).
- Manifest diagnostic snippets include map keys (since 0.68.0).
- Check metadata accepts `examples` (since 0.59.0) and `Minimum Trivy Version`
  (since 0.63.0).
