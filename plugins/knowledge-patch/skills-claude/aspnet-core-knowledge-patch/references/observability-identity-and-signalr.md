# Observability, Identity, and SignalR

Use this reference when upgrading Blazor Identity code, adding passkeys to an existing app, or
instrumenting authentication and ASP.NET Core Identity.

## Passkeys in existing Blazor Web Apps

ASP.NET Core 10 provides a dedicated migration path for adding passkey user authentication to an
existing Blazor Web App (batch `10.0-migration`). Treat this as an Identity migration rather than
assuming passkeys are available only to newly generated projects. Preserve the application's
existing account data and flows while applying the dedicated passkey changes.

## Identity redirects with navigation exceptions disabled

The Blazor Web App template sets the following property to avoid navigation exceptions during
static server-side rendering:

```xml
<BlazorDisableThrowNavigationException>true</BlazorDisableThrowNavigationException>
```

When an older Individual Accounts app opts into this behavior during the `10.0-migration`, edit
`Components/Account/IdentityRedirectManager.cs`:

1. Remove the `InvalidOperationException` thrown by `RedirectTo`.
2. Remove all five `[DoesNotReturn]` attributes.

Those annotations and the explicit throw encode the earlier control-flow contract. Leaving them
in place misrepresents a redirect that can now complete without a navigation exception.

## Authentication metrics

Batch `10.0` adds metrics for authentication duration and for challenge, forbid, sign-in,
sign-out, and authorization counts. Use these built-in instruments before wrapping every handler
solely to count authentication operations.

Correlate counts with failures and duration rather than treating a high event count by itself as
an error.

## Identity metrics

Identity instruments are emitted by the `Microsoft.AspNetCore.Identity` meter. Examples include:

```text
aspnetcore.identity.user.create.duration
aspnetcore.identity.user.check_password_attempts
aspnetcore.identity.sign_in.sign_ins
```

Enable the meter in the application's metrics pipeline, then select instruments according to the
operation being investigated. The names distinguish user-management, password-check, and
sign-in behavior; avoid collapsing all of them into one undifferentiated counter.
