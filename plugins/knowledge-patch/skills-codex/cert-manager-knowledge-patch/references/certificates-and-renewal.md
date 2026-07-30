# Certificates and Renewal

Use this reference for Certificate API behavior, key rotation, renewal timing,
keystore output, subject handling, and issuance safety.

## Private keys and revision history

`spec.privateKey.rotationPolicy` defaults to `Always` from 1.18. Set it to
`Never` on a Certificate that must reuse its prior private key. From 1.20, the
default-Always behavior is GA and cannot be disabled globally through
`DefaultPrivateKeyRotationPolicyAlways`.

`spec.revisionHistoryLimit` defaults to `1` from 1.18. Set an explicit value
when a different number of old CertificateRequests must be retained.

While a Certificate is being deleted, its controller does not create a new
CertificateRequest or Secret (since 1.17). Automation should not expect child
resources to be replenished during finalization.

## Renewal scheduling

The 1.17 correction to `renewBeforePercentage` can shift the computed renewal
time of existing Certificates. The 1.21 implementation also handles durations
longer than approximately three years correctly; previous behavior could reject
them or calculate a wrong renewal time.

The Certificate API adds `renewalPolicies` in 1.21 for more expressive renewal
scheduling alongside `renewBefore` and `renewBeforePercentage`.

Failed CertificateRequests use exponential backoff with a configurable maximum
duration. The default ceiling is 32 hours. Set it through
`--certificate-request-maximum-backoff-duration`, the controller configuration,
or Helm:

```yaml
config:
  certificateRequestMaximumBackoffDuration: 8h
```

If an issuer returns an already-expired certificate, 1.21 stops the response
from causing an infinite reissuance loop. From 1.18.5, a returned certificate
whose public key does not match the CSR is rejected before storage and issuance
fails with backoff instead of looping.

## Keystores and output formats

A Certificate can set `spec.keystores.jks.password` or
`spec.keystores.pkcs12.password` directly (since 1.17). A literal password is
mutually exclusive with that keystore's `passwordSecretRef`. It exists for
software that requires a password and does not improve keystore security.

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

`AdditionalCertificateOutputFormats` is GA in 1.18, so additional formats are
always enabled without a feature gate.

The `Modern2026` PKCS#12 profile added in 1.21 uses AES-256 and SHA-256 KDFs
instead of legacy 3DES or RC2 and is compatible with FIPS 140-3 requirements.

PEM decoding size limits are configurable in 1.20 for certificates or keys
that exceed normal decoder limits. From 1.18.3, cert-manager can parse larger
PEM certificates and chains, including leaves containing many DNS names or
other identities.

## Algorithms and subjects

Signature algorithm selection is configurable from 1.18, allowing a
Certificate to meet a CA or consumer's algorithm requirement.

During the 1.17 transition, RSA key size changes also select stronger hashes:
3072-bit keys use SHA-384 and 4096-bit keys use SHA-512. Confirm consumer
compatibility before rotating affected certificates.

From 1.18, an IP address in `commonName` is put into `ipAddresses` rather than
being incorrectly added to DNS subject alternative names.

Trailing-dot DNS names in X.509 SAN fields were rejected in 1.19.0 because of
a dependency change. Use 1.19.1 or later, which restores support.

## Name constraints and other names

`NameConstraints` became beta and enabled by default in 1.17. URI constraints
require 1.17.4 or later: earlier 1.17 releases incorrectly copied permitted URI
domains into the excluded URI domains of the CSR.

`OtherNames` is beta and enabled by default in 1.20.

## Annotations on generated Certificates

The controller's `--extra-certificate-annotations` option accepts annotation
keys to copy from an Ingress or Gateway to its generated Certificate (since
1.18).

Changes to the Duration or `RenewBefore` annotation on an Ingress or Gateway
immediately update the generated Certificate from 1.20.

Cert-shim also understands `cert-manager.io/alt-names` and
`cert-manager.io/ip-sans` on ingress-like resources from 1.21. See the Gateway
and Ingress reference for the source-resource behavior.

## Validity monitoring

Two gauges added in 1.18 expose a Certificate's issuance and expiration
timestamps:

```text
certmanager_certificate_not_before_timestamp_seconds
certmanager_certificate_not_after_timestamp_seconds
```
