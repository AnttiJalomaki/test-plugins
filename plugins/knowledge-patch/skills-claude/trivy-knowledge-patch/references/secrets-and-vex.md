# Secrets and VEX

## Secret inspection boundaries

- In client/server mode, the configuration analyzer performs secret inspection
  (since 0.60.0).
- Python `.dist-info` directories are ignored during secret scanning
  (since 0.62.0).
- Secret matching requires meaningful secret length and allows example strings
  to remain unflagged (since 0.63.0).
- Input is validated as UTF-8 before protobuf marshalling, and multiline secret
  findings report corrected line numbers (since 0.65.0).
- Skipped folders, files, and extensions can be customized. Azure secret rules
  are available; passwords and passphrases in Maven `settings.xml` and
  `settings-security.xml` are detected; the secret-scanner configuration file
  itself is skipped (since 0.71.0).

## Secret detector formats

- The Symfony default secret key is detected, and Hugging Face tokens use
  improved word-boundary matching (since 0.69.0).
- OpenAI secret rules are included, and stateless GitHub App installation tokens
  are recognized (since 0.72.0).

When tuning exclusions, prefer the specific folder/file/extension mechanism.
Keep UTF-8 validation and location reporting intact in remote scan adapters, and
do not report detector configuration as if it were target content.

## VEX discovery and transport

- A CycloneDX SBOM can reference external VEX documents that Trivy loads during
  scanning (since 0.60.0).
- VEX repositories accept per-repository TLS configuration (since 0.69.0).
- VEX documents stored beneath the scanned repository directory are discovered
  (since 0.72.0).

Resolve referenced and repository-local documents relative to their proper
source. Apply TLS settings to the individual repository rather than globally.

## Dependency graph behavior

- A looping package graph does not cause VEX processing to suppress an
  otherwise applicable vulnerability (since 0.67.0).

Preserve package identities and graph relationships before applying VEX status.
Cycle detection is a traversal concern, not evidence that a finding should be
discarded.
