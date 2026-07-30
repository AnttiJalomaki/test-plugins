# Observability, Identity, and SignalR

## Contents

- [Framework-native OpenTelemetry tracing](#framework-native-opentelemetry-tracing)
- [Handled-exception diagnostics](#handled-exception-diagnostics)
- [Authentication and Identity metrics](#authentication-and-identity-metrics)
- [Passkeys and Identity migration](#passkeys-and-identity-migration)
- [SignalR authentication refresh](#signalr-authentication-refresh)
- [SignalR invocation cancellation](#signalr-invocation-cancellation)
- [Development JWTs for file-based apps](#development-jwts-for-file-based-apps)

## Framework-native OpenTelemetry tracing

ASP.NET Core request activities carry the required OpenTelemetry HTTP server
semantic-convention attributes without
`OpenTelemetry.Instrumentation.AspNetCore` (11.0-preview.2). Subscribe tracing
to the framework activity source:

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddSource("Microsoft.AspNetCore")
        .AddConsoleExporter());
```

Set the
`Microsoft.AspNetCore.Hosting.SuppressActivityOpenTelemetryData` AppContext
switch to `true` only when those attributes must be suppressed.

## Handled-exception diagnostics

Exceptions handled by `IExceptionHandler` do not emit logs or other
diagnostics by default (10.0). Use
`ExceptionHandlerOptions.SuppressDiagnosticsCallback` to select handled
exceptions that should be reported or to restore the earlier behavior.

```csharp
app.UseExceptionHandler(new ExceptionHandlerOptions
{
    SuppressDiagnosticsCallback = context => false
});
```

## Authentication and Identity metrics

ASP.NET Core reports authentication duration and challenge, forbid, sign-in,
sign-out, and authorization counts (10.0).

Identity metrics use the `Microsoft.AspNetCore.Identity` meter. Instruments
include:

- `aspnetcore.identity.user.create.duration`
- `aspnetcore.identity.user.check_password_attempts`
- `aspnetcore.identity.sign_in.sign_ins`

Use those framework instruments before creating duplicate application
measurements for the same operations.

## Passkeys and Identity migration

### Existing Blazor Web Apps

ASP.NET Core provides a dedicated migration path for adding passkey
authentication to an existing Blazor Web App (10.0-migration). Apply that
migration rather than copying only the current template's UI.

### Friendly authenticator names

The Blazor Web App template maps known authenticator AAGUIDs to friendly names
for Google Password Manager, iCloud Keychain, Windows Hello, 1Password, and
Bitwarden (11.0-preview.2). Unknown authenticators still go to the rename
page. Extend `PasskeyAuthenticators.cs` to recognize more AAGUIDs.

### Identity redirect migration

The newer Blazor Web App template sets
`<BlazorDisableThrowNavigationException>true</BlazorDisableThrowNavigationException>`
to avoid navigation exceptions during static SSR (10.0-migration).

When an older Individual Accounts application opts into this setting:

1. Remove the `InvalidOperationException` explicitly thrown by
   `RedirectTo`.
2. Remove all five `[DoesNotReturn]` attributes from
   `Components/Account/IdentityRedirectManager.cs`.

## SignalR authentication refresh

Enable refresh on the mapped hub to expose authentication refresh beside
negotiation (11.0-preview.6):

```csharp
app.MapHub<ChatHub>("/chat", options =>
    options.EnableAuthenticationRefresh = true);
```

The .NET client refreshes before token expiry without dropping the connection
and updates the connection identity. Hubs can override
`OnAuthenticationRefreshedAsync`. Clients tune the default-enabled refresh
behavior with `WithAuthenticationRefresh`.

JavaScript clients and Azure SignalR do not yet support this refresh flow.

## SignalR invocation cancellation

Canceling the `CancellationToken` passed to a regular, non-streaming .NET
client `InvokeAsync` sends cancellation to the server and triggers the hub
method's `CancellationToken` (11.0-preview.6). Client-driven server
cancellation is no longer limited to streaming invocations.

```csharp
using var cts = new CancellationTokenSource();
var work = connection.InvokeAsync("LongRunningWork", cts.Token);
cts.Cancel();
```

## Development JWTs for file-based apps

`dotnet user-jwts` accepts a file-based application with no project file
through `--file` (11.0-preview.6):

```bash
dotnet user-jwts create --file app.cs
```
