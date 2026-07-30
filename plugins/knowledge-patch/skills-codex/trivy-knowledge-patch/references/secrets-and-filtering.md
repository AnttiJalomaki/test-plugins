# Secrets and Filtering

## Client/server inspection

- In client/server mode, the configuration analyzer performs secret inspection
  (since 0.60.0).
- Secret input is checked for valid UTF-8 before protobuf marshalling, and
  multiline matches report corrected line numbers (since 0.65.0).

## Default exclusions and false-positive controls

- Secret scanning ignores Python `.dist-info` directories (since 0.62.0).
- Matches must have a meaningful length, and example strings can remain
  unflagged (since 0.63.0).
- Skipped folders, files, and extensions are configurable. The secret-scanner
  configuration file itself is not scanned (since 0.71.0).

## Detector coverage

- The Symfony default secret key is detected, and Hugging Face token matching
  uses improved word boundaries (since 0.69.0).
- Azure secret rules are included. Passwords and passphrases in Maven
  `settings.xml` and `settings-security.xml` are detected (since 0.71.0).
- OpenAI secret rules are included, and stateless GitHub App installation token
  formats are recognized (since 0.72.0).

