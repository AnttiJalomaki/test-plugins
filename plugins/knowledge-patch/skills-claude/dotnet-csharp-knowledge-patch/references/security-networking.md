# Security, cryptography, and networking

## Cryptography migration checks

.NET 10 introduces several compatibility changes (`10.0-guides`):

- Composite ML-DSA moved to draft-08.
- ML-DSA and SLH-DSA renamed `SecretKey` members to `PrivateKey`.
- `CoseSigner.Key` and key parameters on `X509Certificate` and `PublicKey`
  can be null.
- Unix requires OpenSSL 1.1.1 or later.
- OpenSSL cryptographic primitives are unsupported on macOS.
- `X500DistinguishedName` validation is stricter.
- The OpenSSL override variable is `DOTNET_OPENSSL_VERSION_OVERRIDE`.

On macOS, DSA is unavailable in .NET 11 Preview 6
(`11.0-preview.6-compatibility`). Replace DSA-based protocols and keys before
upgrading.

## Certificate lookup and PFX export

`X509Certificate2Collection.FindByThumbprint` accepts a `HashAlgorithmName`,
avoiding the SHA-1-only behavior and same-length-hash ambiguity of
`Find(FindByThumbprint, ...)` (`10.0`).

`X509Certificate.ExportPkcs12` accepts a `Pkcs12ExportPbeParameters` preset or
custom `PbeParameters`. Choose deliberately between broadly compatible
3DES/SHA-1 output and modern AES-256/SHA-256 output.

```csharp
var matches = certificates.FindByThumbprint(
    HashAlgorithmName.SHA256, thumbprint);
byte[] pfx = certificate.ExportPkcs12(
    Pkcs12ExportPbeParameters.Pbes2Aes256Sha256, password);
```

## Post-quantum cryptography

`MLKem`, `MLDsa`, and `SlhDsa` use static key-generation and import methods
instead of deriving from `AsymmetricAlgorithm`. Check the relevant
`IsSupported` property before use: availability requires OpenSSL 3.5 or later,
or Windows CNG with post-quantum support.

```csharp
if (MLKem.IsSupported)
{
    using var key = MLKem.GenerateKey(MLKemAlgorithm.MLKem768);
}
```

`MLDsa` and `SlhDsa` remain experimental under `SYSLIB5006`; some `MLKem`
methods are also experimental.

## AES KeyWrap with Padding

`Aes` implements RFC 5649 AES-KWP through APIs such as
`DecryptKeyWrapPadded`. When writing to a caller-provided destination, this
method returns the unwrapped key length.

```csharp
using Aes aes = Aes.Create();
aes.SetKey(keyEncryptionKey);
int length = aes.DecryptKeyWrapPadded(wrappedKey, destination);
```

## X25519 key exchange

.NET 11 Preview 6 adds `X25519DiffieHellman.GenerateKey()`
(`11.0-preview.6`). It selects the platform implementation and supports raw
keys plus PKCS#8, SubjectPublicKeyInfo, and PEM import and export.

## TLS and certificate-chain behavior

### macOS client TLS 1.3

.NET 10 macOS clients can opt into TLS 1.3 for `SslStream` and `HttpClient` by
using Network.framework:

```csharp
AppContext.SetSwitch("System.Net.Security.UseNetworkFramework", true);
```

The equivalent environment variable is
`DOTNET_SYSTEM_NET_SECURITY_USENETWORKFRAMEWORK=1`. This path is client-only.
It can remove TLS 1.0 and 1.1 availability and alter buffering, cancellation,
zero-byte reads, and internationalized-domain-name behavior.

### Server-side AIA downloads

`SslStream` disables server-side Authority Information Access certificate
downloads by default in .NET 11 Preview 6. Server validation must not assume
missing chain certificates will be fetched; provide chain material or an
explicit validation strategy.

## HTTP, QUIC, URI, and mail behavior

.NET 10 networking defaults and validation changes include:

- Trimmed publications disable HTTP/3 by default.
- Browser HTTP clients stream responses by default.
- `Uri` no longer applies its former length limits.
- `MailAddress` rejects addresses containing consecutive dots.

In .NET 11 Preview 6, `HttpClient` automatically falls back to HTTP/1.1 when
NTLM or Negotiate Windows authentication cannot work over HTTP/2.
`QuicStream.Priority` exposes a priority from 0, highest, to 255, lowest;
`DefaultPriority` is 127.
