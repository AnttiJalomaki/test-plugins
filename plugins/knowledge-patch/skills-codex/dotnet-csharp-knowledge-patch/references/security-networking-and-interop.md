# Security, Networking, and Interop

Use this reference for certificates, cryptographic algorithms, TLS, HTTP,
QUIC, address validation, LDAP, native-library resolution, and COM. It
contains guidance from `10.0-guides`, `10.0`,
`11.0-preview.6-compatibility`, and `11.0-preview.6`.

## Certificate lookup and PFX export

`X509Certificate2Collection.FindByThumbprint` accepts a
`HashAlgorithmName`. Prefer it over SHA-1-only
`Find(FindByThumbprint, ...)`; the explicit algorithm also avoids ambiguity
between same-length hashes.

```csharp
var matches = certificates.FindByThumbprint(
    HashAlgorithmName.SHA256,
    thumbprint);
```

`X509Certificate.ExportPkcs12` accepts a `Pkcs12ExportPbeParameters` preset or
custom `PbeParameters`. This allows an intentional choice between broadly
compatible 3DES/SHA-1 and modern AES-256/SHA-256 output:

```csharp
byte[] pfx = certificate.ExportPkcs12(
    Pkcs12ExportPbeParameters.Pbes2Aes256Sha256,
    password);
```

Check the target importer before choosing the modern preset.

## Post-quantum cryptography

`MLKem`, `MLDsa`, and `SlhDsa` expose static key-generation and import methods
rather than deriving from `AsymmetricAlgorithm`.

Always check each type's `IsSupported` property:

```csharp
if (MLKem.IsSupported)
{
    using var key = MLKem.GenerateKey(MLKemAlgorithm.MLKem768);
}
```

Availability requires OpenSSL 3.5 or later, or Windows CNG with PQC support.
`MLDsa` and `SlhDsa` remain experimental under `SYSLIB5006`, and some
`MLKem` methods are experimental (`10.0`).

Compatibility details from `10.0-guides`:

- Composite ML-DSA moved to draft-08.
- ML-DSA and SLH-DSA `SecretKey` members were renamed to `PrivateKey`.

Do not persist or exchange draft-specific encodings without an explicit
format/version strategy.

## Symmetric key wrapping

`Aes` implements RFC 5649 AES KeyWrap with Padding. Methods such as
`DecryptKeyWrapPadded` can write into a caller-provided destination and return
the unwrapped key length:

```csharp
using Aes aes = Aes.Create();
aes.SetKey(keyEncryptionKey);
int length = aes.DecryptKeyWrapPadded(wrappedKey, destination);
```

Use only the returned prefix of the destination.

## X25519

`X25519DiffieHellman.GenerateKey()` selects the platform implementation
(`11.0-preview.6`). The API supports raw keys and PKCS#8,
SubjectPublicKeyInfo, and PEM import/export, providing a first-party X25519
key-agreement path.

## Cryptography compatibility

The following `10.0-guides` changes can affect source code, validation, or
deployment:

- `CoseSigner.Key` can be null.
- Key parameters on `X509Certificate` and `PublicKey` can be null.
- Unix requires OpenSSL 1.1.1 or later.
- OpenSSL cryptographic primitives are unsupported on macOS.
- `X500DistinguishedName` validation is stricter.
- The OpenSSL override variable is `DOTNET_OPENSSL_VERSION_OVERRIDE`.

In `11.0-preview.6-compatibility`, DSA is unavailable on macOS.

Treat nullable key properties as an explicit state and validate them before
use. Re-test distinguished names received from external systems.

## TLS

### Opt-in macOS client TLS 1.3

On macOS, clients can opt into Network.framework for TLS 1.3 in `SslStream`
and `HttpClient`:

```csharp
AppContext.SetSwitch(
    "System.Net.Security.UseNetworkFramework",
    true);
```

The environment alternative is
`DOTNET_SYSTEM_NET_SECURITY_USENETWORKFRAMEWORK=1`.

This path is client-only. It can remove TLS 1.0/1.1 availability and can
change buffering, cancellation, zero-byte-read, and IDN behavior. Exercise
protocol and cancellation tests under the opted-in implementation.

### Server-side certificate chains

In `11.0-preview.6-compatibility`, `SslStream` disables server-side AIA
certificate downloads by default. Servers must not assume missing
intermediate chain material will be fetched through AIA; supply a complete
chain or configure validation deliberately.

## HTTP and QUIC

Compatibility defaults from `10.0-guides`:

- trimmed publications disable HTTP/3 by default;
- browser HTTP clients stream responses by default; and
- `Uri` no longer imposes its former length limits.

Code that buffered browser responses implicitly should opt into the needed
behavior or consume streams correctly. Apply application-level URI limits at
trust boundaries if size constraints are required.

In `11.0-preview.6`, `HttpClient` falls back automatically to HTTP/1.1 when
NTLM or Negotiate Windows authentication cannot operate over HTTP/2.

`QuicStream.Priority` ranges from 0 (highest) to 255 (lowest);
`DefaultPriority` is 127.

## Address and LDAP validation

`MailAddress` rejects addresses containing consecutive dots
(`10.0-guides`). LDAP `DirectoryControl` parsing is also stricter. Treat new
failures as input incompatibilities and correct the source data or protocol
encoding instead of bypassing validation.

## Native-library resolution

Single-file apps no longer probe the executable directory for native
libraries (`10.0-guides`). Package native dependencies or configure an
explicit supported resolution path.

`DllImportSearchPath.AssemblyDirectory` searches only the assembly directory;
it no longer implies a broader executable-directory search. Re-test
deployment layouts in which managed and native artifacts are separated.

## COM reflection

Casting an `IDispatchEx` COM object to `IReflect` now fails
(`10.0-guides`). Use the supported COM dispatch surface rather than relying on
that cast.
