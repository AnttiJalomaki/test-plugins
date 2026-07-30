# Images, Registries, and Operating Systems

## Image and registry acquisition

- Registry mirrors can supply mirrored endpoints for container image scans
  (since 0.59.0).
- GHCR artifact downloads use `GITHUB_TOKEN` for authentication (since
  0.59.0).
- Images whose total layer size exceeds the configured limit are rejected
  early rather than scanned further (since 0.59.0).
- Unsupported remote artifact types are rejected instead of being processed as
  container images (since 0.64.0).
- Root.io container images are supported (since 0.64.0). Root.io package
  matching checks the full package version and uses corrected vulnerability
  severity selection (since 0.65.0).
- Docker contexts are resolved when locating an image to scan (since 0.65.0).
- AWS image acquisition supports dual-stack ECR endpoints (since 0.68.0).
- ActiveState images can be scanned for vulnerabilities (since 0.69.0).

## Image metadata and history

- Image reports save layer metadata for downstream report consumers (since
  0.62.0).
- Image-history scanning does not run check `AVD-DS-0007` (since 0.60.0).
- `CreatedBy` values from Buildah and legacy Docker-builder histories are
  normalized before instruction checks (since 0.64.0).
- During image misconfiguration analysis, `.Config.User` always takes
  precedence over `USER` entries in `.History` (since 0.64.0).
- Build-metadata suffixes are removed from image history, and legacy `ENV`
  values are quoted so spaces survive parsing (since 0.67.0).
- Docker archive analysis preserves `RepoTags` (since 0.68.0).
- Embedded-SBOM images produce deterministic scan results (since 0.69.0).
- `RUN` instructions are reconstructed correctly for images built without
  BuildKit (since 0.71.0).
- Custom resources are attributed to their origin layer after image layers are
  merged (since 0.72.0).

## OS selection and lifecycle

- `--distro` manually selects the OS distribution when automatic detection is
  unavailable or inappropriate (since 0.59.0).
- Distribution aliases map to their corresponding operating systems (since
  0.60.0).
- Bottlerocket package analysis and MinimOS detection are supported (since
  0.63.0).
- Lifecycle data includes EOL dates for RHEL 10, Ubuntu 25.04, and Ubuntu 20.04
  ESM (since 0.64.0).
- AlmaLinux 10 is supported, and the Alma `rpmqa` parser accepts epoch-qualified
  versions (since 0.65.0).
- RHEL-derived images can be detected through `os-release` (since 0.67.0).
- Photon 5.0 vulnerability scanning is supported (since 0.68.0).
- Rocky Linux modular packages participate in vulnerability matching (since
  0.69.0).
- Unsupported vulnerability detection for the CentOS Stream family is skipped
  without failing the complete scan (since 0.69.0).
- Ubuntu 25.10 lifecycle data is available (since 0.70.0), and OS detection
  recognizes Ubuntu 26.04 LTS (since 0.71.0).
- When the detected OS is overridden, OS package PURLs are updated to match the
  replacement identity (since 0.71.0).
- Bottlerocket packages can be matched against vulnerability data (since
  0.72.0).

## OS package metadata and matching

- Duplicate `dpkg` packages found at different paths in different image layers
  are consolidated (since 0.59.0).
- Debian `dpkg` results omit empty license values (since 0.61.0).
- Alpine APK analysis extracts package maintainer metadata (since 0.63.0).
- Red Hat analysis searches the root layer for build information, preserves
  `contentSets` for OS packages in filesystem and VM scans, and removes invalid
  suffixes from manifest `content_sets` (since 0.63.0).
- Wolfi scanning recognizes the newer APK database location (since 0.63.0).
- Debian packages are classified as third-party from a maintainer list.
  Distribution vulnerability data is not applied to third-party Debian or
  Ubuntu packages (since 0.69.0).
- The common vulnerability detection path skips any package identified as
  third-party (since 0.70.0).
- `ospkg.NewScanner` forwards detector options to OS package detectors (since
  0.72.0).

