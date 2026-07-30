# Artifacts and Registries

## Registry and remote image acquisition

- Container image acquisition supports registry mirrors (since 0.59.0).
- GHCR artifact downloads honor `GITHUB_TOKEN` (since 0.59.0).
- `trivy registry login` authenticates without requesting a registry scope
  (since 0.60.0).
- Remote retrieval rejects unsupported artifact types instead of attempting to
  scan them as container images (since 0.64.0).
- Image resolution follows Docker contexts when locating local images
  (since 0.65.0).
- AWS image acquisition supports dual-stack ECR endpoints (since 0.68.0).

Keep authentication failures, unsupported-artifact failures, and scan failures
distinct. A mirror or resolved Docker context can change the concrete image
source even when the user-facing reference is unchanged.

## Image size, layers, and history

- Image scans reject images whose total layer size exceeds the limit and return
  an early error (since 0.59.0).
- Duplicate `dpkg` packages at different paths in different layers are
  consolidated (since 0.59.0).
- Image scans retain layer metadata in reports (since 0.62.0).
- Buildah and legacy Docker `CreatedBy` history values are normalized so
  instruction-based misconfiguration checks see consistent commands
  (since 0.64.0).
- `.Config.User` takes precedence over `USER` instructions in `.History` when
  image misconfiguration scanning determines the effective user
  (since 0.64.0).
- Image history normalization strips build-metadata suffixes and quotes legacy
  `ENV` values so embedded spaces survive (since 0.67.0).
- Dockerfile analysis tolerates unsupported experimental flags and maps the
  health-check start-period option to `--start-period` (since 0.68.0).
- `RUN` instructions are reconstructed correctly for images built without
  BuildKit (since 0.71.0).
- After image layers are merged, custom resources are attributed back to their
  origin layer (since 0.72.0).

## Archives, attestations, and embedded SBOMs

- Docker archive analysis preserves `RepoTags` (since 0.68.0).
- Image and SBOM handling accepts SBOMs carried in Sigstore bundles and SPDX
  attestations (since 0.68.0).
- Images that contain an embedded SBOM produce deterministic scan results
  (since 0.69.0).
- Image scan results can carry image layer data originating in an SBOM
  (since 0.59.0).

## Artifact and repository identity

- Repository scans add Git repository metadata to reports (since 0.65.0).
- Filesystem cache hits retain `RepoMetadata`, and repository URLs are sanitized
  before being written to report metadata (since 0.66.0).
- A filesystem scan whose target contains Git information classifies the
  artifact as a repository, changing the artifact type visible to report
  consumers (since 0.68.0).
- Scan targets expose a unique `ArtifactID` whose calculation includes registry
  and repository; reports expose a UUIDv7 `ReportID`, vulnerability
  fingerprints, and the image reference in report metadata (since 0.68.0).
- SARIF reports use the correct `ROOTPATH` URI for Git repository targets
  (since 0.70.0).

Do not key external state only by an unqualified image name or filesystem path.
Preserve repository classification, registry/repository inputs, IDs, layer
metadata, and sanitized source metadata through intermediate adapters.

## Filesystem analysis boundaries

- Filesystem scans invoke post-analyzers for static paths, and `--file-patterns`
  applies to every post-analyzer (since 0.61.0).
- Python secret scans ignore `.dist-info` directories (since 0.62.0), while
  ordinary SBOM discovery excludes PEP 770 documents under `.dist-info/sboms/`
  (since 0.69.0).
