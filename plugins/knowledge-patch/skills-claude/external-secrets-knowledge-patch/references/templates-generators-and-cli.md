# Templates, Generators, and CLI

Use this reference for rendering behavior, template functions, credential
generators, CLI workflows, and platform release artifacts.

## Template evaluation and values

- Templates accept non-standard delimiters, which avoids collisions with secret
  content or another embedded template language that uses the defaults (0.15.0).
- Value-scope processing preserves the native value instead of coercing it to a
  string (0.19.0).
- Values loaded through `templateFrom` are decoded before template evaluation
  (2.7.0).
- Generic target paths preserve mixed-case components (2.7.0).
- Slice notation resolves correctly in the parser (2.7.0).
- Environment is included when group variables are selected (0.18.0).

## Template helpers and removals

- `certSANs` extracts subject alternative names from a certificate (2.2.0).
- `hexdec` converts hexadecimal input to decimal (2.7.0).
- `getHostByName` was removed (2.3.0). Remove DNS lookups from templates and
  provide the resolved value through a controlled input instead.

## Metadata and source processing

- `result.jsonpath` in a `dataFrom` request can itself be templated (0.18.0).
- Secret metadata can request decoded values explicitly (0.15.0).
- Secret templates can define finalizers on generated Secrets (0.20.0).
- A configurable null-byte policy applies to sources (2.3.0).

The API and lifecycle consequences of these features are covered in
`api-and-reconciliation.md`.

## Registry credential generators

- Quay is available as a generator source (0.13.0).
- Cloudsmith can generate container-registry authentication credentials
  (0.20.0).
- GitLab deploy-token generation is available (2.8.0).
- The AWS ECR authorization-token generator accepts custom ECR endpoints
  (0.18.0) and resolves credentials through the AWS credential chain (0.19.0).

## Grafana service-account generator

A Grafana service-account generator is available (0.14.0). Its in-cluster
integration passes the requested role when creating the service account
(0.15.0). `SecondsToLive` is optional, so a manifest need not set an explicit
token lifetime (2.8.0).

## SSH key generator

The SSH key generator was added in 0.19.0 and supports ECDSA keys as of 1.1.0.
Validate key type and downstream parser support when rotating from another key
algorithm.

## MFA token generator

An MFA token generator is available (0.18.0). Its length option is optional, and
the same release corrected length handling. Omit the option for the default or
set it deliberately; do not retain a workaround for the earlier behavior.

## Bootstrap generators and validation

`esoctl` provides bootstrap-generator commands (1.0.0). Generator references
validate `externalsecret_type` (1.0.0), so a mistyped or mismatched reference is
rejected rather than passed through to reconciliation.

The `STSSessionToken` generator removed its JWT-token authentication option in
0.19.0. Migrate those generator configurations to another supported
authentication path.

## Rendering with esoctl

The template-data and secret renderer is exposed through `esoctl` (0.13.0). The
release action and installed executable use the name `esoctl`, not `render`.
Use it to evaluate template inputs before applying a resource when debugging
delimiters, native values, decoding, functions, or source selection.

## Release binaries

Native `darwin_arm64` artifacts are available for Apple Silicon macOS (1.1.0).
For container-image signatures, provenance, and SBOM attestations, see
`security-and-support.md`.
