# PKI, Transit, and Managed Keys

## Issuer and chain enforcement

- PKI enforces issuer extended-key-usage, name-constraint, and issuer-name
  extensions when issuing or signing leaf certificates.
- A signing CA's `max_path_length` limits `root/sign-intermediate`.
- Unusable issuers reject manually configured chains. Enterprise issuer fields
  can disable selected chain validations.
- Enterprise Common Criteria mode validates the full chain, can enable
  validation-time checks, treats `NotBefore` as zero, enforces minimum key
  usages for extended-key-usage sets, and rejects uploaded certificates
  without a chain of trust.

## Expiry, subjects, and CSR handling

- `leaf_not_after_behavior = "always_enforce_err"` rejects an excessive TTL
  for leaf, CA, and ACME issuance.
- Roles accept `serial_number_source`; issuance observes the configured maximum
  TTL.
- Root and intermediate generation accepts the remaining permitted and
  excluded email, IP, URI, and DNS name-constraint fields.
- Role `alt_names` accepts glob-style DNS names.
- PKI `sign-verbatim` copies a CSR basic-constraints extension when
  `isCA=false` and rejects it when `isCA=true`.
- The legacy role field `allow_token_displayname` is deprecated. Replace it
  with domain, bare-domain, subdomain, or glob-domain constraints before its
  targeted April 2027 removal.

## CRLs and certificate responses

- `max_crl_entries` prevents an excessive revocation list from overloading
  Vault.
- Mount- and issuer-level AIA can advertise Freshest CRLs. Base CRLs include
  the Freshest CRL extension.
- Issue, sign, and fetch responses include certificate `AuthorityKeyID`.
- Enterprise `batch/certs` fetches multiple certificates.
- Enterprise `integrations/guardium` configures the Guardium integration.

## ACME, SCEP, EST, and CMPv2

- ACME `challenge_permitted_ip_ranges` and
  `challenge_excluded_ip_ranges` constrain HTTP-01 and TLS-ALPN-01 validation
  destinations.
- ACME APIs list account key IDs, retrieve account, order, and certificate
  details, and update account status. Validation failures have ACME-specific
  error types.
- Enterprise PKI provides a SCEP server, including issuers backed by RSA
  PKCS#11 managed keys.
- SCEP roles enforce `token_bound_cidrs`. `GetCACaps` reports configured
  encryption and digest algorithms and advertises `POSTPKIOperation`.
- SCEP, EST, and CMPv2 can issue certificates without `server_flag` key usage.
- Enterprise PKI External CA obtains public-CA certificates through ACME.
  Vault Agent can execute these workflows and re-renders templates after
  issuance or renewal.

## Transit algorithms and workflows

- Transit supports experimental ML-DSA and SLH-DSA signatures. Reload fixes
  keep stored ML-DSA and SLH-DSA keys usable after policy reload.
- Transit RSA encryption supports `pkcs1v15` padding.
- Enterprise supports Ed25519ph and Ed25519ctx signing and verification.
- Enterprise Transit supports AES-CBC encryption/decryption and 192-bit AES
  CMAC keys.
- `context` on datakeys and derived-keys endpoints encrypts derived DEKs.
- Rewrap supports managed keys.
- Transit envelope encryption lets applications perform bulk encryption
  locally while Vault protects their data-encryption keys.
- Core and Transit random-byte APIs allow larger responses and pseudorandom
  output seeded from random sources. Large requests consume proportionally
  more memory.

## TLS, FIPS, key sizes, and entropy

- FIPS builds use a FIPS 140-3 module and compliant algorithms.
- The Go TLS stack supports X25519MLKEM768 hybrid post-quantum key agreement.
- Enterprise KMIP RSA generation enforces a 2048-bit minimum.
- SSH secrets caps RSA keys at 8192 bits from 1.19.18.
- Enterprise password policies can select an entropy source, including `seal`
  for entropy augmentation.
- Enterprise `common_criteria_mode` constrains listener TLS cipher suites.

## Managed keys, SSH, KMIP, and KMS

- Enterprise GCP managed keys support workload identity federation.
- Enterprise SSH secrets can sign SSH keys with managed keys.
- `GET sys/managed-keys/:type/:name` returns named usages: `encrypt`,
  `decrypt`, `sign`, `verify`, `wrap`, `unwrap`, `generate_random`, and `mac`.
- KMIP APIs manage multiple client-verification CAs and import external CAs.
  Enterprise also has an experimental endpoint for executing KMIP requests.
- Enterprise key management supports multi-region AWS KMS keys.
