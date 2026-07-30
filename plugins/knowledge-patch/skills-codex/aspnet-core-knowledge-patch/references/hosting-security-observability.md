# Hosting, Security, Compression, and Observability

## Diagnostics and metrics

- ASP.NET Core emits authentication duration plus challenge, forbid, sign-in, sign-out, and authorization counts. Identity instruments use the `Microsoft.AspNetCore.Identity` meter, including `aspnetcore.identity.user.create.duration`, `aspnetcore.identity.user.check_password_attempts`, and `aspnetcore.identity.sign_in.sign_ins` (10.0).
- Exceptions handled by an `IExceptionHandler` no longer produce logs and other diagnostics by default. Set `ExceptionHandlerOptions.SuppressDiagnosticsCallback` to choose what is reported; return `false` from the callback to restore reporting for handled exceptions (10.0).
- Request activities include the OpenTelemetry HTTP server semantic-convention attributes without `OpenTelemetry.Instrumentation.AspNetCore`. Subscribe with `.AddSource("Microsoft.AspNetCore")`; set the `Microsoft.AspNetCore.Hosting.SuppressActivityOpenTelemetryData` AppContext switch to `true` to suppress the attributes (11.0-preview.2).

## Kestrel, HTTP.sys, and request handling

- Kestrel treats configured `*.localhost` hosts as loopback bindings rather than wildcard external bindings. The `web` and `blazor` templates accept `--localhost-tld` for names such as `<project>.dev.localhost`; retrust the .NET 10 development certificate with `dotnet dev-certs https --trust` so it covers `*.dev.localhost` (10.0).
- `HttpSysOptions.RequestQueueSecurityDescriptor` accepts a `GenericSecurityDescriptor` that grants or denies queue access to users and groups. It applies only when HTTP.sys creates a new request queue and cannot modify an existing queue (10.0).
- ASP.NET Core registers `IMemoryPoolFactory<byte>`; `Create` returns pools that evict idle blocks automatically. Replacing the factory does not add eviction unless the custom factory implements it (10.0).
- `ITlsHandshakeFeature.Exception` exposes the original failed-handshake exception after connection middleware continues. Replace obsolete `HttpsConnectionAdapterOptions.TlsClientHelloBytesCallback` with `UseTlsClientHelloListener`, registered before `UseHttps` (11.0-preview.4).
- Kestrel preserves `%2F` in HTTP/1.1 absolute-form paths, matching origin-form behavior. `GET http://host/a%2Fb` resolves to `/a%2Fb`, not `/a/b`; update routing or middleware that relied on decoding (11.0-preview.4).
- `Limits.RequestHeadersTimeout` now covers incomplete, fragmented HTTP/2 and HTTP/3 trailer header blocks (11.0-preview.5).

## Caching and compression

- `IOutputCachePolicyProvider` can provide default base policies through `IReadOnlyList<IOutputCachePolicy> GetBasePolicies()` and resolve named policies dynamically through `ValueTask<IOutputCachePolicy?> GetPolicyAsync(string policyName)`, including tenant- or external-configuration policies (11.0-preview.1).
- Response compression and request decompression support zstd and enable it by default. `ZstandardCompressionOptions.Quality` ranges from 1 to 22; higher values trade speed for compression ratio (11.0-preview.3).
- Response-compression middleware emits `Vary: Accept-Encoding` even when it does not compress the response, preventing shared caches from mixing variants (11.0-preview.4).

## Browser request protection and development credentials

- Apps created with `WebApplication.CreateBuilder` reject unsafe cross-origin browser form requests automatically using `Sec-Fetch-Site` and `Origin`. Same-origin requests, user navigations, and non-browser clients remain allowed. Opt out per endpoint with `.DisableAntiforgery()` or `[IgnoreAntiforgeryToken]`, app-wide with `DisableCsrfProtection`, or replace trust decisions with `ICsrfProtection`. Blazor Web Apps no longer need `UseAntiforgery()` (11.0-preview.6).
- In WSL, `dotnet dev-certs https --trust` installs and trusts the development certificate in both WSL and Windows (11.0-preview.1).
- `dotnet user-jwts create --file app.cs` issues development JWTs for file-based apps that have no project file (11.0-preview.6).

## SignalR

- Set `EnableAuthenticationRefresh = true` on a mapped hub to advertise refresh beside negotiation. The .NET client refreshes before token expiry without dropping the connection and updates its identity; hubs may override `OnAuthenticationRefreshedAsync`, and clients can tune the default-enabled behavior with `WithAuthenticationRefresh`. JavaScript clients and Azure SignalR are not supported yet (11.0-preview.6).
- Canceling the token passed to a non-streaming .NET client `InvokeAsync` sends cancellation to the server and triggers the hub method's `CancellationToken`; cancellation is no longer limited to streaming calls (11.0-preview.6).
