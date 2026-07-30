# Certificates and Issuance

Use this reference for Certificate fields, renewal scheduling, generated
keystores, signing behavior, name handling, and issuance safety.

## Private keys and revision history

### Rotation defaults to `Always`

Since `upgrade-1.18`, an omitted `Certificate.spec.privateKey.rotationPolicy`
means `Always`, not `Never`. Set `Never` explicitly before upgrading when a
consumer requires the existing private key:

```yaml
spec:
  privateKey:
    rotationPolicy: Never
```

The `DefaultPrivateKeyRotationPolicyAlways` behavior became GA in `1.20`, and
its feature gate is no longer configurable. Explicit per-Certificate policy is
the supported way to retain a key.

### Revision history is bounded by default

Since `upgrade-1.18`, an omitted `Certificate.spec.revisionHistoryLimit`
defaults to `1` instead of `nil`. Set an explicit larger limit if operational
or audit workflows need more historical CertificateRequests.

## Renewal scheduling

### Percentage calculations

The `renewBeforePercentage` calculation was corrected in `1.17` to follow its
API specification. An upgrade can therefore change the renewal time of an
existing Certificate that uses this field.

In `1.21`, percentage renewal was also corrected for Certificate durations
longer than roughly three years. Earlier behavior could reject such values or
compute the wrong renewal time.

### Expressive renewal policies

The Certificate API in `1.21` adds `renewalPolicies` for more expressive
scheduling alongside `renewBefore` and `renewBeforePercentage`. Choose one
coherent policy and validate its interaction with the issuer's validity
period.

### Annotation changes reconcile immediately

In `1.20`, changing the Duration or `RenewBefore` annotation on an Ingress or
Gateway API resource immediately triggers an update of its generated
Certificate. Do not rely on an unrelated source-resource change to force
reconciliation.

## Keystores and output formats

### Literal keystore passwords

Since `1.17`, JKS and PKCS#12 output can use a literal password at
`spec.keystores.jks.password` or `spec.keystores.pkcs12.password`. Each literal
field is mutually exclusive with its corresponding `passwordSecretRef`.
Literal passwords exist for software compatibility and do not add substantive
keystore security.

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example
spec:
  secretName: example-tls
  issuerRef:
    name: example-issuer
  keystores:
    jks:
      create: true
      password: changeit
  dnsNames:
    - example.com
```

### Additional output is unconditional

`AdditionalCertificateOutputFormats` is GA in `1.18`. Additional certificate
output formats no longer require a feature gate.

### FIPS-compatible PKCS#12

The `Modern2026` profile in `1.21` produces PKCS#12 output using AES-256 and
SHA-256 KDFs rather than legacy 3DES or RC2, making it suitable for FIPS 140-3
requirements.

## Names, constraints, and signing

### Retire the `ValidateCAA` gate

`ValidateCAA` was deprecated in `upgrade-1.17` with removal planned for 1.18.
Stop manually enabling it and remove it from feature-gate configuration.

### Name constraints lifecycle and fixes

`NameConstraints` became beta and enabled by default in `1.17`, allowing CA
certificates to express name constraints without manual gate enablement.

Use 1.17.4 or later for URI name constraints. Earlier 1.17 patches incorrectly
copied permitted URI domains into the excluded URI domains of the CSR.

### IP common names and SANs

Since `1.18`, a `commonName` that is an IP address is placed into
`ipAddresses`, rather than being incorrectly added to DNS subject alternative
names.

### Trailing-dot DNS SANs

Version 1.19.0 rejected DNS names ending in a trailing dot in X.509 SAN fields.
Version 1.19.1 restored support; use 1.19.1 or later where fully qualified SANs
retain their trailing dot.

### Other names

The `OtherNames` feature is beta and enabled by default in `1.20`.

### Signature algorithms and large RSA keys

The signature algorithm is configurable since `1.18`, allowing issuance to
meet a CA or consumer's algorithm requirements.

Since `upgrade-1.17`, certificates with 3072-bit RSA keys use SHA-384 and those
with 4096-bit RSA keys use SHA-512. If larger-key certificates fail when they
rotate, confirm that every consumer supports the stronger hash algorithm.

## Issuance and reconciliation safety

### No new children during deletion

Since `1.17`, a Certificate being deleted does not cause its controller to
create new CertificateRequest or Secret objects.

### Larger PEM objects

Starting in 1.18.3, cert-manager accepts larger PEM certificates and chains,
including leaf certificates containing large numbers of DNS names or other
identities. In `1.20`, PEM decoding size limits became operator-configurable
for certificates or keys that exceed normal decoder limits.

### Reject mismatched public keys

Starting in 1.18.5, cert-manager rejects an issued certificate whose public key
does not match its CSR. Issuance fails with backoff before the certificate is
stored, instead of entering an infinite reissuance loop.

### Reject already-expired responses

In `1.21`, an already-expired certificate returned by an issuer stops with a
failure rather than triggering an infinite reissuance loop.

### CertificateRequest retry ceiling

In `1.21`, the maximum exponential-backoff duration for a failed
CertificateRequest is configurable and defaults to 32 hours. Configure it with
`--certificate-request-maximum-backoff-duration`, a controller configuration
file, or the Helm value:

```yaml
config:
  certificateRequestMaximumBackoffDuration: 8h
```
