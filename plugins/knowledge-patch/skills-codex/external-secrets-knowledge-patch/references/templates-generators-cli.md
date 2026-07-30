# Templates, generators, and CLI

## Template evaluation and values

- Configure non-standard delimiters when secret content or an embedded
  template language collides with the defaults (since 0.15.0).
- Value-scoped processing preserves native values instead of coercing them to
  strings (since 0.19.0).
- `result.jsonpath` can be templated in `dataFrom`, allowing the extraction
  path to be selected dynamically (since 0.18.0).
- Values loaded through `templateFrom` are decoded before use (since 2.7.0).
- Generic target paths preserve mixed-case components (since 2.7.0).
- Slice notation resolves correctly in the template parser (since 2.7.0).
- Secret metadata can request decoded-value encoding (since 0.15.0).
- Group-variable selection accounts for environments (since 0.18.0).

## Template functions

- `certSANs` extracts subject alternative names from certificates (since
  2.2.0).
- `hexdec` converts hexadecimal input to decimal (since 2.7.0).
- `getHostByName` was removed; templates that depend on DNS lookup through it
  must be rewritten (since 2.3.0).

## Template output behavior

- Secret templates can add finalizers to generated Secrets (since 0.20.0).
- PKCS#12 processing supports bundles that contain certificates but no private
  key (since 0.20.0).
- Template `objectMeta` and `ownerReferences` propagate to target resources
  (since 2.3.0).

## Credential generators

- Quay is supported as a generator source (since 0.13.0).
- Grafana service-account credentials can be generated (since 0.14.0). The
  generator works with in-cluster Grafana and passes the requested role when
  creating the service account (since 0.15.0). `SecondsToLive` is optional
  (since 2.8.0).
- The MFA token generator has an optional length and correct length handling
  (since 0.18.0).
- SSH key generation is available (since 0.19.0) and supports ECDSA keys (since
  1.1.0).
- Cloudsmith registry credentials can be generated (since 0.20.0).
- GitLab deploy tokens can be generated (since 2.8.0).
- AWS ECR authorization-token generation supports custom endpoints (since
  0.18.0) and resolves credentials through the AWS credential chain (since
  0.19.0).
- The `STSSessionToken` generator has no JWT-token authentication option; use
  another supported authentication path (since 0.19.0).

## Generator validation and processing

- `generatorRef` validates `externalsecret_type` (since 1.0.0).
- The chart's `processClusterGenerator` boolean controls whether
  cluster-scoped generators are processed (since 0.20.0).

## Rendering and esoctl

A renderer for template data and Secrets is available. The release action is
named `esoctl`, not `render` (since 0.13.0). `esoctl` also exposes bootstrap
generator commands (since 1.0.0).

## Release and custom-build tooling

- Native `darwin_arm64` release artifacts support Apple Silicon macOS (since
  1.1.0).
- Every provider has a build tag, allowing unwanted providers to be disabled
  in custom builds (since 1.1.0).
