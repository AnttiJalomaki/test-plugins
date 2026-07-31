# Security, Networking, and Interop

The items here are organized by operational concern. They originate from the
`10.0-guides` and `10.0` batches.

## Certificate Lookup by Thumbprint Algorithm

`X509Certificate2Collection.FindByThumbprint` accepts a
`HashAlgorithmName`. Prefer it when looking up a SHA-256 or other non-SHA-1
thumbprint:

```csharp
var matches = certificates.FindByThumbprint(
    HashAlgorithmName.SHA256, thumbprint);
```

This avoids the SHA-1-only behavior and same-length hash ambiguity of the
older `Find(FindByThumbprint, ...)` path.

## Explicit PKCS#12 Export Encryption

`X509Certificate.ExportPkcs12` accepts either a
`Pkcs12ExportPbeParameters` preset or custom `PbeParameters`:

```csharp
byte[] pfx = certificate.ExportPkcs12(
    Pkcs12ExportPbeParameters.Pbes2Aes256Sha256, password);
```

Choose deliberately between broadly compatible 3DES/SHA-1 output and modern
AES-256/SHA-256 output. The strongest output is not necessarily readable by
every legacy consumer.

## Post-Quantum Cryptography

`MLKem`, `MLDsa`, and `SlhDsa` use static generation and import methods; they
do not derive from `AsymmetricAlgorithm`. Check each algorithm type's
`IsSupported` property before use. Availability requires OpenSSL 3.5 or later,
or Windows CNG with post-quantum support.

```csharp
if (MLKem.IsSupported)
{
    using var key = MLKem.GenerateKey(MLKemAlgorithm.MLKem768);
}
```

`MLDsa` and `SlhDsa` remain experimental under `SYSLIB5006`, and some `MLKem`
methods are also experimental. Composite ML-DSA follows draft-08. The
`SecretKey` members on ML-DSA and SLH-DSA were renamed to `PrivateKey`; update
source and serialized member mappings that used the old name.

## AES KeyWrap with Padding

`Aes` implements RFC 5649 AES-KWP. Span-based methods such as
`DecryptKeyWrapPadded` write to caller-provided storage and return the actual
unwrapped length:

```csharp
using Aes aes = Aes.Create();
aes.SetKey(keyEncryptionKey);
int length = aes.DecryptKeyWrapPadded(wrappedKey, destination);
```

Use the returned length when slicing or consuming the destination buffer.

## Cryptography Compatibility Checklist

- `CoseSigner.Key` may be null. Key parameters on `X509Certificate` and
  `PublicKey` may also be null. Preserve the nullable contract instead of
  blindly dereferencing them.
- Unix cryptography requires OpenSSL 1.1.1 or later.
- OpenSSL cryptographic primitives are not supported on macOS. Choose native
  supported paths there.
- `X500DistinguishedName` validation is stricter; reject or repair malformed
  distinguished names at the boundary.
- The OpenSSL override environment variable is
  `DOTNET_OPENSSL_VERSION_OVERRIDE`. Replace older variable names in launch
  scripts and deployment manifests.

## macOS Client TLS 1.3

`SslStream` and `HttpClient` can opt in to Network.framework for client-side
TLS 1.3. Set either the AppContext switch or environment variable:

```csharp
AppContext.SetSwitch("System.Net.Security.UseNetworkFramework", true);
```

```text
DOTNET_SYSTEM_NET_SECURITY_USENETWORKFRAMEWORK=1
```

The opt-in is client-only. It can remove TLS 1.0 and 1.1 availability and can
change buffering, cancellation, zero-byte-read, and internationalized domain
name behavior. Exercise those behaviors before enabling it broadly.

## HTTP and Browser Defaults

Trimmed publications disable HTTP/3 by default. Enable it deliberately only
when the trimmed deployment contains and supports the required pieces.

Browser HTTP clients stream responses by default. Code that relied on complete
buffering before a response was returned must request or implement that
behavior explicitly.

## URI and Mail Validation

`Uri` no longer enforces its former length limits. Applications needing a
resource-exhaustion or business limit must enforce one at their own boundary.

`MailAddress` rejects addresses containing consecutive dots. Treat rejection
as input validation rather than attempting to preserve the old permissive
parsing.

## Native Library Loading

Single-file applications no longer probe the executable directory for native
libraries. Package native assets correctly or configure an explicit resolver
or search location.

`DllImportSearchPath.AssemblyDirectory` searches only the assembly directory.
It no longer implies a broader fallback. Re-test plugins and single-file
deployments whose managed assembly and native dependency live in different
directories.

## COM Reflection

Casting an `IDispatchEx` COM object to `IReflect` now fails. Use the COM
dispatch surface or an explicit adapter; do not use `IReflect` as an assumed
projection for an `IDispatchEx` object.
