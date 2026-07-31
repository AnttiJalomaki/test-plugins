# Security, Cryptography, and Networking

## Cryptography compatibility (`10.0-guides`)

### Post-quantum key and format changes

- Composite ML-DSA uses the draft-08 format.
- The `SecretKey` members on ML-DSA and SLH-DSA were renamed to `PrivateKey`.
- `CoseSigner.Key` may be null. Key parameters on `X509Certificate` and
  `PublicKey` may also be null; handle absence explicitly.

### OpenSSL and certificate validation

- Unix requires OpenSSL 1.1.1 or later.
- OpenSSL cryptographic primitives are not supported on macOS.
- `X500DistinguishedName` validation is stricter. Revalidate names that older
  runtimes accepted.
- The OpenSSL override environment variable is
  `DOTNET_OPENSSL_VERSION_OVERRIDE`.

## Networking compatibility (`10.0-guides`)

- Trimmed publications disable HTTP/3 by default. Enable it deliberately when
  the deployment needs it.
- Browser HTTP clients stream responses by default. Code that needs complete
  buffering must request or implement it explicitly.
- `Uri` no longer applies its former length limits. Apply an application-level
  bound where untrusted input must remain bounded.
- `MailAddress` rejects addresses containing consecutive dots.
- LDAP `DirectoryControl` parsing is stricter; test existing encoded controls
  and reject malformed input rather than relying on permissive parsing.

## Cryptographic APIs (`10.0`)

### Algorithm-specific certificate lookup

`X509Certificate2Collection.FindByThumbprint` accepts a
`HashAlgorithmName`. Use it to avoid the SHA-1-only behavior and same-length
hash ambiguity of `Find(FindByThumbprint, ...)`.

```csharp
var matches = certificates.FindByThumbprint(
    HashAlgorithmName.SHA256, thumbprint);
```

### Configurable PFX export

`X509Certificate.ExportPkcs12` accepts a `Pkcs12ExportPbeParameters` preset or
custom `PbeParameters`. Select broadly compatible 3DES/SHA-1 output only when a
consumer requires it; otherwise the AES-256/SHA-256 preset provides the modern
choice.

```csharp
byte[] pfx = certificate.ExportPkcs12(
    Pkcs12ExportPbeParameters.Pbes2Aes256Sha256, password);
```

### Post-quantum cryptography APIs

`MLKem`, `MLDsa`, and `SlhDsa` use static key generation and import methods;
they do not derive from `AsymmetricAlgorithm`. Check each type's `IsSupported`
property because availability requires OpenSSL 3.5+ or Windows CNG with
post-quantum support. `MLDsa` and `SlhDsa` are experimental under `SYSLIB5006`,
and some `MLKem` methods are experimental.

```csharp
if (MLKem.IsSupported)
{
    using var key = MLKem.GenerateKey(MLKemAlgorithm.MLKem768);
}
```

### AES KeyWrap with Padding

`Aes` implements RFC 5649 AES-KWP through methods including
`DecryptKeyWrapPadded`. When writing into a caller-provided destination, the
method returns the unwrapped key length.

```csharp
using Aes aes = Aes.Create();
aes.SetKey(keyEncryptionKey);
int length = aes.DecryptKeyWrapPadded(wrappedKey, destination);
```

## macOS client TLS 1.3 (`10.0`)

Opt `SslStream` and `HttpClient` clients into TLS 1.3 through
Network.framework by setting the `System.Net.Security.UseNetworkFramework`
AppContext switch or `DOTNET_SYSTEM_NET_SECURITY_USENETWORKFRAMEWORK=1`.

```csharp
AppContext.SetSwitch("System.Net.Security.UseNetworkFramework", true);
```

This switch is client-only. It can remove TLS 1.0/1.1 and change buffering,
cancellation, zero-byte-read, and IDN behavior, so exercise those paths before
deploying it.
