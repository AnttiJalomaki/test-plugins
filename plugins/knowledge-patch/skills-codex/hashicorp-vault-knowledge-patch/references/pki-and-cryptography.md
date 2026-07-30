# PKI and cryptography

Use this reference for PKI issuance and validation, ACME and enrollment protocols, Transit algorithms and workflows, managed keys, KMIP, FIPS, Common Criteria, and cryptographic resource limits.

## Transit algorithms

- Transit adds experimental ML-DSA signatures. Later fixes ensure stored ML-DSA and SLH-DSA keys remain usable after their policies reload.
- Experimental SLH-DSA signatures are available from 1.20.
- Enterprise supports Ed25519ph and Ed25519ctx signing and verification.
- RSA encryption accepts `pkcs1v15` padding.
- Enterprise supports 192-bit keys for AES CMAC (since 1.20).
- Enterprise adds AES-CBC encryption and decryption (since 1.21).

## Transit workflows

- The datakeys and derived-keys endpoints accept `context` for encrypting derived data-encryption keys (Enterprise, since 1.21).
- Rewrap supports managed keys (Enterprise, since 1.21).
- Transit supports envelope encryption from 2.0: applications encrypt or decrypt bulk data locally while Vault protects the data-encryption keys.
- Core and Transit random-byte APIs allow larger responses and can return pseudorandom bytes seeded from random sources. Large requests consume proportionally more memory.

## TLS, FIPS, and asymmetric bounds

- FIPS builds use a FIPS 140-3 cryptographic module and compliant algorithms.
- The Go TLS stack supports X25519MLKEM768 hybrid post-quantum key agreement.
- Enterprise KMIP RSA generation enforces a minimum size of 2048 bits.
- The SSH secrets engine caps RSA keys at 8192 bits from 1.19.18.

## Issuer validation

- PKI enforces issuer extended-key-usage, name-constraint, and issuer-name extensions when issuing or signing leaf certificates.
- A signing CA's `max_path_length` constrains `root/sign-intermediate`.
- Manual chains reject unusable issuers. Enterprise issuer fields can disable selected chain validations where an exception is explicitly required.
- `leaf_not_after_behavior = "always_enforce_err"` rejects TTLs extending past the issuer even for CA issuance and ACME.
- Issuance respects the configured maximum TTL.

## Subject names and role constraints

- Roles accept `serial_number_source`.
- Root and intermediate creation supports permitted and excluded email, IP, URI, and DNS name constraints.
- Role `alt_names` accept glob-style DNS names.
- `allow_token_displayname` is deprecated and targeted for removal in April 2027. Replace it with constraints such as `allowed_domains`, `allow_bare_domains`, `allow_subdomains`, or `allow_glob_domains` (`upgrade-safety`).

## Chains and Common Criteria mode

Enterprise `common_criteria_mode`:

- restricts listener TLS cipher suites;
- validates the full certificate chain;
- can enable validation-time checks;
- treats `NotBefore` as zero;
- enforces minimum key usages for extended-key-usage sets; and
- rejects uploaded certificates that lack a chain of trust.

Test both issuance and uploaded-certificate paths before enabling it.

## CRLs and distribution points

- Set `max_crl_entries` to prevent an unbounded CRL from exhausting Vault resources.
- Mount- and issuer-level AIA data can advertise Freshest CRLs.
- Base CRLs include the Freshest CRL extension (since 1.20).

## ACME validation and administration

- `challenge_permitted_ip_ranges` and `challenge_excluded_ip_ranges` constrain HTTP-01 and TLS-ALPN-01 validation targets.
- APIs can list account key IDs, fetch account, order, and certificate detail, and update account status.
- ACME validation failures return ACME-specific error types.
- The Enterprise PKI External CA plugin can acquire public-CA certificates through ACME (since 2.0).
- Vault Agent can execute those ACME workflows natively, and Agent templates re-render when an external-CA certificate is issued or renewed.

## SCEP, EST, and CMPv2

- SCEP, EST, and CMPv2 can issue certificates without the `server_flag` key usage.
- Enterprise PKI exposes a SCEP server for clients that do not use Vault APIs (since 1.20).
- SCEP can use an issuer backed by an RSA PKCS#11 managed key.
- Enterprise SCEP roles enforce `token_bound_cidrs` (since 1.21).
- `GetCACaps` reflects configured encryption and digest algorithms and advertises `POSTPKIOperation`.

## PKI response and integration APIs

- Issue, sign, and fetch responses include the certificate `AuthorityKeyID` (since 1.21).
- Enterprise `batch/certs` fetches multiple certificates.
- Enterprise `integrations/guardium` configures the Guardium integration.
- `sign-verbatim` copies a CSR basic-constraints extension when `isCA=false` and rejects it when `isCA=true`.

## Managed keys and KMIP

- Enterprise GCP managed keys accept workload identity federation credentials.
- `GET sys/managed-keys/:type/:name` returns usage names rather than integer IDs: `encrypt`, `decrypt`, `sign`, `verify`, `wrap`, `unwrap`, `generate_random`, and `mac` (since 2.0). Update typed clients and decoders.
- Enterprise SSH secrets can use managed keys to sign SSH keys (since 1.20).
- KMIP APIs manage multiple client-verification CAs and import external CAs (since 2.0).
- Enterprise also has an experimental API for executing KMIP requests.
- The Enterprise key-management secrets engine supports multi-region AWS KMS keys.

## Password-generation entropy

Enterprise password-generation policies can select an entropy source, including `seal` for entropy augmentation (since 1.21).

## MSSQL external key management

The MSSQL EKM provider lets administrators choose which Transit key versions wrap and unwrap SQL Server data-encryption keys. This supports restoring encrypted backups that depend on a specific wrapping-key version (since 1.21).

## PKI review checklist

1. Resolve edition, feature maturity, issuer, role, key type, and backing managed key.
2. Validate name constraints, EKUs, path length, chain usability, TTL, and requested subject data together.
3. Bound CRL size and ACME validation networks.
4. Confirm protocol capabilities for SCEP, EST, CMPv2, External CA, and Agent automation.
5. Update clients for named managed-key usages and `AuthorityKeyID` response data.
6. Load-test memory-sensitive random-byte and revocation workflows.
