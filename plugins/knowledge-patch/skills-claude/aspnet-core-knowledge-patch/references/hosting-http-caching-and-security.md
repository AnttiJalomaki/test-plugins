# Hosting, HTTP, Caching, and Security

Use this reference for browser HTTP behavior, server diagnostics, development hosts, request
parsing, pooled memory, and HTTP.sys request queues. These changes are from batch `10.0`.

## Streaming `HttpClient` responses in Blazor WebAssembly

Response streaming is enabled by default. `ReadAsStreamAsync` returns a
`BrowserHttpReadStream`, not a buffered `MemoryStream`, and the browser stream does not support
synchronous reads. Keep downstream deserialization and copying asynchronous.

Disable streaming per request when a dependency requires a seekable or synchronously readable
buffer:

```csharp
requestMessage.SetBrowserResponseStreamingEnabled(false);
```

Temporary global opt-outs are available through either control:

```xml
<WasmEnableStreamingResponse>false</WasmEnableStreamingResponse>
```

```text
DOTNET_WASM_ENABLE_STREAMING_RESPONSE=0
```

Prefer the per-request control when only one integration is incompatible.

## Exception-handler diagnostic suppression

An exception handled by `IExceptionHandler` no longer emits logs and other diagnostics by
default. Use `ExceptionHandlerOptions.SuppressDiagnosticsCallback` to decide which handled
exceptions remain observable.

```csharp
app.UseExceptionHandler(new ExceptionHandlerOptions
{
    SuppressDiagnosticsCallback = context => false
});
```

Returning `false` restores diagnostics for the matching exception. Align this callback with
logging and tracing policy so expected business errors remain quiet while operational failures
remain visible.

## `.localhost` development domains

Kestrel treats configured `*.localhost` hosts as loopback bindings rather than wildcard external
bindings. The `web` and `blazor` templates accept `--localhost-tld` and can use a domain such as
`<project>.dev.localhost`.

The ASP.NET Core 10 development certificate covers `*.dev.localhost`, but the updated certificate
must be trusted again:

```bash
dotnet dev-certs https --trust
```

Recheck launch URLs and callback URLs after changing the template domain.

## `PipeReader`-based JSON deserialization

MVC, Minimal APIs, and `ReadFromJsonAsync` deserialize JSON through `PipeReader`. A custom
converter cannot assume `Utf8JsonReader.ValueSpan` contains the complete token because
`HasValueSequence` may be `true`.

```csharp
var span = reader.HasValueSequence
    ? reader.ValueSequence.ToArray()
    : reader.ValueSpan;
```

Audit converters that read raw token bytes, especially converters tested only with small,
single-segment payloads. The following AppContext switch temporarily restores stream-based
parsing during migration:

```text
Microsoft.AspNetCore.UseStreamBasedJsonParsing
```

Set it to `true`; do not treat the switch as a substitute for sequence-safe converter code.

## Evicting memory pools from dependency injection

ASP.NET Core registers `IMemoryPoolFactory<byte>`. Its `Create` method returns memory pools that
automatically evict idle blocks. Resolve the factory when framework-aligned eviction is desired.

A custom `IMemoryPoolFactory<byte>` registration does not gain idle-block eviction merely by
implementing the interface. Its implementation must supply equivalent eviction behavior.

## HTTP.sys request-queue security descriptors

Set `HttpSysOptions.RequestQueueSecurityDescriptor` to a `GenericSecurityDescriptor` to grant or
deny request-queue access for users and groups. The descriptor is applied only when HTTP.sys
creates a new request queue; it cannot modify an existing queue. Verify queue lifecycle before
relying on an ACL change, and recreate the queue through an appropriate operational procedure
when a changed descriptor must take effect.
