# Hosting, Security, Compression, and Observability

These hosting and diagnostics behaviors apply to ASP.NET Core `10.0`.

## Authentication and Identity metrics

ASP.NET Core reports authentication duration and counts for challenge,
forbid, sign-in, sign-out, and authorization operations. Identity metrics use
the `Microsoft.AspNetCore.Identity` meter. Available instruments include:

- `aspnetcore.identity.user.create.duration`
- `aspnetcore.identity.user.check_password_attempts`
- `aspnetcore.identity.sign_in.sign_ins`

Use the actual meter and instrument names when configuring collection or
dashboards.

## Exception-handler diagnostic suppression

An exception handled by `IExceptionHandler` no longer emits logs or other
diagnostics by default. Configure `SuppressDiagnosticsCallback` to report
selected handled exceptions or return `false` to restore the earlier behavior:

```csharp
app.UseExceptionHandler(new ExceptionHandlerOptions
{
    SuppressDiagnosticsCallback = context => false
});
```

## `.localhost` development domains

Kestrel treats configured `*.localhost` hosts as loopback bindings, not
wildcard external bindings. The `web` and `blazor` templates accept
`--localhost-tld` and can use a host such as `<project>.dev.localhost`.

The development certificate covers `*.dev.localhost` after it is trusted
again:

```console
dotnet dev-certs https --trust
```

## Evicting memory pools from dependency injection

ASP.NET Core registers `IMemoryPoolFactory<byte>`. Its `Create` method returns
pools that automatically evict idle blocks. Replacing the factory through
dependency injection removes that guarantee unless the custom implementation
provides equivalent eviction itself.

## HTTP.sys request-queue security descriptors

Set `HttpSysOptions.RequestQueueSecurityDescriptor` to a
`GenericSecurityDescriptor` to grant or deny request-queue access for users
and groups. The setting applies only when HTTP.sys creates a new request queue;
it cannot modify an existing queue.

## Test top-level-statement applications

The ASP.NET Core source generator emits the `public partial class Program`
needed by test projects. Remove a manual declaration from applications that
use top-level statements to avoid maintaining a redundant compatibility shim.
