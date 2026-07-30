# CLI, Reports, and Runtime

## CLI flags and configuration

- Vulnerability scans accept `--distro` to override automatic OS distribution
  detection when detection is unavailable or unsuitable (since 0.59.0).
- `--generate-default-config` omits hidden flags from generated configuration
  (since 0.59.0).
- Use `--vuln-severity-source` to select the vulnerability severity source
  (since 0.60.0), for example:

  ```sh
  trivy image --vuln-severity-source nvd alpine:3.20
  ```

- Reports can include a summary table (since 0.60.0).
- `trivy registry login` no longer sends a registry scope during authentication
  (since 0.60.0).
- JSONC configuration inputs accept comments and trailing commas (since
  0.63.0).
- The CLI can check for available Trivy versions (since 0.63.0).
- The `sbom` command disables `--skip-dir` and `--skip-files`; remove these
  flags from SBOM-command automation (since 0.63.0).
- `--compliance` is not restricted to a predefined allowed-values list (since
  0.63.0).
- The `--server` value is schema-validated and invalid values fail flag
  validation (since 0.65.0).
- `--list-all-pkgs` defaults to `true`; explicitly use
  `--list-all-pkgs=false` for the earlier selective output (since 0.67.0).
- Configuration-only options distinguish user-supplied values from defaults
  (since 0.68.0).
- `trivy cloud` is supported, and `--cacert` supplies a CA certificate for
  custom trust (since 0.68.0).
- A JSON Schema for `trivy.yaml` enables validation and editor support;
  array-valued enums are represented on the schema item definition (since
  0.69.0).
- Template flag validation rejects files with invalid template extensions
  (since 0.70.0).

## Scan targets and Kubernetes output

- Kubernetes scanning supports controllers, and `--report all` emits the
  complete requested report (since 0.61.0).
- Kubernetes artifact comparison uses the correct artifact versions and no
  longer reads the `last-applied-configuration` annotation (since 0.62.0).
- Kubernetes summary reports omit passed misconfiguration checks (since
  0.62.0).
- Filesystem scans run post-analyzers for static paths, and `--file-patterns`
  applies to every post-analyzer (since 0.61.0).
- When Git information is detected, a filesystem artifact is classified as a
  repository, which changes its reported artifact type (since 0.68.0).

## Report fields and formatting

- SARIF leaves `shortDescription` and `fullDescription` text unescaped rather
  than HTML-escaping it (since 0.60.0).
- Image reports retain layer metadata for downstream consumers (since 0.62.0).
- JUnit templates include source locations for misconfiguration findings
  (since 0.63.0).
- Git repository metadata is included in repository-scan reports (since
  0.65.0).
- SARIF vulnerability findings include CVSS vectors (since 0.65.0).
- Filesystem cache hits preserve `RepoMetadata` (since 0.66.0).
- Repository URLs are sanitized before being added to report metadata (since
  0.66.0).
- Scan targets expose a unique `ArtifactID` calculated partly from registry and
  repository. Reports expose a UUIDv7 `ReportID`, vulnerability fingerprints,
  and the image reference in report metadata (since 0.68.0).
- Detected packages include `AnalyzedBy`, identifying the analyzer that found
  them (since 0.69.0).
- JSON reports include the Trivy version that produced them (since 0.69.0).
- Client/server JSON reports include the server version. The server `/version`
  response excludes JavaDB and CheckBundle entries (since 0.70.0).
- Azure and Mariner detectors populate the detected-vulnerability fields in
  results (since 0.70.0).
- SARIF reports use the correct `ROOTPATH` URI for Git repository targets
  (since 0.70.0).
- The `gitlab.tpl` link array has no trailing comma (since 0.71.0).
- When vulnerability details are unavailable, results use `UNKNOWN` severity
  (since 0.72.0).

## Runtime, server, cache, and plugins

- Termination signals trigger graceful shutdown, while normal exits do not log
  the graceful-shutdown message (since 0.65.0).
- HTTP request/response tracing is supported, and server mode creates an HTTP
  transport (since 0.65.0).
- Updating plugin `index.yaml` preserves existing plugins (since 0.66.0).
- If vulnerability database contents are absent while metadata remains, Trivy
  downloads the database again rather than accepting the metadata-only cache
  (since 0.67.0).
- The vulnerability database supports concurrent access by multiple scan
  operations (since 0.68.0).

