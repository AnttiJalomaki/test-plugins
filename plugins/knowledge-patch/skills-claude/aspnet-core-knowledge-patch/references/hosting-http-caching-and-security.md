# Hosting, HTTP, Caching, and Security

## Contents

- [Development hosts and certificates](#development-hosts-and-certificates)
- [Cross-origin CSRF protection](#cross-origin-csrf-protection)
- [Compression and cache variance](#compression-and-cache-variance)
- [Dynamic output-cache policies](#dynamic-output-cache-policies)
- [Kestrel protocol behavior](#kestrel-protocol-behavior)
- [TLS handshake hooks](#tls-handshake-hooks)
- [HTTP.sys queue security](#httpsys-queue-security)
- [Evicting memory pools](#evicting-memory-pools)

## Development hosts and certificates

### `.localhost` domains

Kestrel treats configured `*.localhost` hosts as loopback bindings rather than
wildcard external bindings (10.0). The `web` and `blazor` templates accept
`--localhost-tld` to produce names such as `<project>.dev.localhost`.

The development certificate includes `*.dev.localhost`; trust it again after
moving to that certificate:

```bash
dotnet dev-certs https --trust
```

### WSL trust

In WSL, `dotnet dev-certs https --trust` installs and trusts the development
certificate in both WSL and Windows (11.0-preview.1).

## Cross-origin CSRF protection

Applications created with `WebApplication.CreateBuilder` automatically reject
unsafe cross-origin browser requests that consume forms
(11.0-preview.6). The default trust decision uses `Sec-Fetch-Site` and
`Origin`. Same-origin requests, user navigations, and non-browser clients
remain allowed.

Available opt-outs are:

- `.DisableAntiforgery()` on an endpoint.
- `[IgnoreAntiforgeryToken]`.
- The application-wide `DisableCsrfProtection` setting.

Implement `ICsrfProtection` to replace the trust policy. Blazor Web Apps no
longer need to call `UseAntiforgery()`.

## Compression and cache variance

### Zstandard

Response-compression and request-decompression middleware support zstd and
enable it by default (11.0-preview.3). Quality ranges from 1 through 22;
higher values improve compression at the cost of speed.

```csharp
builder.Services.AddResponseCompression();
builder.Services.AddRequestDecompression();
builder.Services.Configure<ZstandardCompressionProviderOptions>(options =>
{
    options.CompressionOptions = new ZstandardCompressionOptions
    {
        Quality = 6
    };
});
```

### Shared-cache variants

When response compression is enabled, the middleware emits
`Vary: Accept-Encoding` even for responses it does not compress
(11.0-preview.4). Preserve that header so shared caches do not mix compressed
and uncompressed representations.

## Dynamic output-cache policies

Custom `IOutputCachePolicyProvider` implementations can provide default base
policies and resolve named policies dynamically, including policies from
tenant-specific or external configuration (11.0-preview.1).

```csharp
public interface IOutputCachePolicyProvider
{
    IReadOnlyList<IOutputCachePolicy> GetBasePolicies();
    ValueTask<IOutputCachePolicy?> GetPolicyAsync(string policyName);
}
```

## Kestrel protocol behavior

### Encoded slashes

For absolute-form HTTP/1.1 request targets, Kestrel preserves `%2F` in the path
as it does for origin-form targets (11.0-preview.4). For example,
`GET http://host/a%2Fb` resolves to `/a%2Fb`, not `/a/b`. Update routing or
middleware that relied on decoding the encoded segment.

### Trailer header timeout

`KestrelServerLimits.RequestHeadersTimeout` applies to incomplete, fragmented
HTTP/2 and HTTP/3 trailer header blocks (11.0-preview.5). Do not add a separate
unbounded trailer path that defeats this connection-lifetime protection.

## TLS handshake hooks

After connection middleware continues from a failed TLS handshake,
`ITlsHandshakeFeature.Exception` exposes the original exception
(11.0-preview.4).

`HttpsConnectionAdapterOptions.TlsClientHelloBytesCallback` is obsolete.
Register `UseTlsClientHelloListener` before `UseHttps`:

```csharp
listenOptions.UseTlsClientHelloListener(
    (connection, clientHelloBytes) => { });
listenOptions.UseHttps();
```

## HTTP.sys queue security

Set `HttpSysOptions.RequestQueueSecurityDescriptor` to a
`GenericSecurityDescriptor` to grant or deny request-queue access for users
and groups (10.0). It applies only when HTTP.sys creates a new request queue;
it cannot change an existing queue.

## Evicting memory pools

ASP.NET Core registers an `IMemoryPoolFactory<byte>` whose `Create` method
returns pools with automatic eviction of idle blocks (10.0). A custom
registered factory does not inherit eviction behavior; it must implement that
behavior itself.
