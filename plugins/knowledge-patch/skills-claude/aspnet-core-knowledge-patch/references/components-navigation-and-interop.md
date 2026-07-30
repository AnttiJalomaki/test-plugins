# Components, Navigation, and Interop

## Contents

- [Rendering and component metadata](#rendering-and-component-metadata)
- [Navigation and routing](#navigation-and-routing)
- [Startup, dependency injection, and browser options](#startup-dependency-injection-and-browser-options)
- [JavaScript and HTTP interop](#javascript-and-http-interop)
- [Circuits and reconnection](#circuits-and-reconnection)
- [Virtualization and grids](#virtualization-and-grids)
- [Culture and localization startup](#culture-and-localization-startup)
- [Render-mode boundaries](#render-mode-boundaries)

## Rendering and component metadata

### Environment-aware rendering

Use `EnvironmentView` to conditionally render content based on the current
environment. `Include` and `Exclude` accept case-insensitive, comma-separated
names and follow MVC environment-tag-helper semantics on the server and in
WebAssembly. The component was introduced as `EnvironmentBoundary` in
11.0-preview.1 and renamed to `EnvironmentView` in 11.0-preview.6.

```razor
<EnvironmentView Include="Development,Staging">
    <DeveloperPanel />
</EnvironmentView>
```

### Request-derived base paths

In Blazor Web Apps, place `<BasePath />` in the document head to render the
base path from `NavigationManager.BaseUri`; it falls back to `/`. Standalone
WebAssembly apps still need a static `<base href="/" />` (11.0-preview.1).

### Interactive MathML

Dynamically rendered `<math>` elements are created in the MathML namespace, so
ordinary MathML markup works during interactive rendering
(11.0-preview.1).

### Custom property injection

Implement `IComponentPropertyActivator.GetActivator(Type)` to control how
component `[Inject]` properties are populated. This supports custom
containers, context-aware resolution, and other injection policies
(11.0-preview.1).

## Navigation and routing

### Same-page navigation

`NavigationManager.NavigateTo` preserves scroll position when the destination
is on the current page, including query-string-only and fragment-only changes
(10.0).

### `NavLinkMatch.All`

`NavLinkMatch.All` compares only the path; query strings and fragments no
longer make an otherwise matching path inactive (10.0). To restore the former
comparison, set this AppContext switch:

```text
Microsoft.AspNetCore.Components.Routing.NavLink.EnableMatchAllForQueryStringAndFragment
```

### Current-URI-relative navigation

Set `NavigationOptions.RelativeToCurrentUri = true` with
`NavigationManager.NavigateTo`, or `RelativeToCurrentUri="true"` on
`NavLink`, to resolve a relative destination against the current page rather
than the application base URI (11.0-preview.1).

### Not Found routing

Call `NavigationManager.NotFound()` to set a 404 during static SSR or signal
the router during interactive rendering. Set `Router.NotFoundPage` to the
routed component and subscribe to `NavigationManager.OnNotFound` for custom
handling. The old router `<NotFound>` fragment is unsupported (10.0).

```razor
<Router AppAssembly="@typeof(Program).Assembly"
        NotFoundPage="typeof(Pages.NotFound)">
    <Found Context="routeData">
        <RouteView RouteData="@routeData" />
    </Found>
</Router>
```

`NavigationManager.GetUriWithHash` was renamed to
`NavigationManager.GetUriWithFragment` in 11.0-preview.6.

## Startup, dependency injection, and browser options

### Unified JavaScript startup shape

`blazor.server.js` and `blazor.webassembly.js` accept the nested `circuit` and
`webAssembly` option objects used by `blazor.web.js`. Their original top-level
shapes remain supported (11.0-preview.1).

```javascript
Blazor.start({
  circuit: {
    reconnectionOptions: { retryIntervalMilliseconds: [0, 2000] }
  }
});
```

### Server-defined browser options

Use `WithBrowserOptions` to serialize startup configuration from C# for
Server, WebAssembly, and Auto render modes, including logging, reconnection,
SSR DOM preservation, and the WebAssembly environment (11.0-preview.6).

```csharp
app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode()
    .WithBrowserOptions(options =>
    {
        options.Server.ReconnectionMaxRetries = 10;
        options.Ssr.PreserveDom = true;
    });
```

Preview migrations:

- Rename `WithBrowserConfiguration` to `WithBrowserOptions`.
- Replace negatively valued `DisableDomPreservation` with positively valued
  `PreserveDom`.
- Replace millisecond-valued `CircuitInactivityTimeoutMs` with the
  `TimeSpan`-valued `CircuitInactivityTimeout`.

### Interactive Server connection configuration

`ServerComponentsEndpointOptions.ConfigureConnection` exposes
`HttpConnectionDispatcherOptions` while Interactive Server render mode is
configured (11.0-preview.1):

```csharp
app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode(options =>
    {
        options.ConfigureConnection = connection =>
            connection.CloseOnAuthenticationExpiration = true;
    });
```

### Hosted services and configuration in WebAssembly

WebAssembly starts and stops registered `IHostedService` implementations with
the browser application's lifetime. Register them with
`AddHostedService<T>()` (11.0-preview.1).

Environment variables are included automatically in WebAssembly
configuration alongside `appsettings.json`, so values such as
`builder.Configuration["API_ENDPOINT"]` are available at runtime
(11.0-preview.1).

## JavaScript and HTTP interop

### JavaScript object construction and properties

`IJSRuntime` and `IJSObjectReference` support
`InvokeConstructorAsync`, `GetValueAsync`, and `SetValueAsync`. In-process
references provide synchronous equivalents (10.0).

```csharp
var instance = await JSRuntime.InvokeConstructorAsync(
    "jsInterop.TestClass", "Blazor!");
var text = await instance.GetValueAsync<string>("text");
await instance.SetValueAsync("text", "updated");
```

### Streaming WebAssembly responses

Response streaming is enabled by default, and `ReadAsStreamAsync` returns
`BrowserHttpReadStream`, not `MemoryStream`. The browser stream does not
support synchronous reads (10.0).

Disable streaming per request:

```csharp
requestMessage.SetBrowserResponseStreamingEnabled(false);
```

Disable it globally with
`<WasmEnableStreamingResponse>false</WasmEnableStreamingResponse>` or
`DOTNET_WASM_ENABLE_STREAMING_RESPONSE=0`.

## Circuits and reconnection

### Reconnection state

The template `ReconnectModal` collocates CSS and JavaScript rather than
injecting styles, making it compatible with strict CSP `style-src` policies.
State changes dispatch `components-reconnect-state-changed`; the state set
includes `retrying` (10.0).

### Circuit state resumption

Interactive Server circuit state can survive an extended disconnect or a
proactive pause and resume without discarding unsaved state. A full-page
refresh still loses that circuit state (10.0).

### Server-initiated pausing

Call `await circuit.RequestCircuitPauseAsync()` to request a graceful client
pause. The result says whether the request reached a connected, initialized
circuit (11.0-preview.4).

There is no active-circuit registry. Capture `Circuit` instances in
`CircuitHandler.OnConnectionUpAsync`, remove them in
`OnCircuitClosedAsync`, and optionally let the client defer a server pause
through the `Blazor.start` option `onPauseRequested`.

## Virtualization and grids

### Variable-height items

`Virtualize<TItem>` measures and adapts to different item heights at runtime
instead of assuming uniform heights (11.0-preview.3). Its default
`OverscanCount` changed from 3 to 15 to improve average-height estimates;
`QuickGrid` continues to use 3.

### Viewport anchoring

`AnchorMode` controls compensation when content above the viewport changes
height (11.0-preview.4):

- `Beginning` is the default and pins the beginning edge.
- `End` follows appends while the user remains at the bottom.
- `None` disables edge pinning.
- The flags may be combined to anchor both edges.

Provide `ItemComparer` when refreshed reference-type items need key-based
identity.

```razor
<Virtualize Items="@messages" AnchorMode="VirtualizeAnchorMode.End">
    <ItemContent Context="message">@message.Text</ItemContent>
</Virtualize>
```

### Indexed scrolling

Set `InitialIndex` for the starting position and call `ScrollToIndexAsync` to
move later (11.0-preview.6). Calling before the first interactive render
throws. Out-of-range indexes are clamped; a newer call or user scroll
supersedes an in-flight scroll.

### Static SSR `QuickGrid`

`QuickGrid` sorts and paginates during static SSR by using enhanced forms and
storing state in the query string, producing refreshable and shareable links
without a circuit or WebAssembly (11.0-preview.5). This is enabled by default.

Set
`Microsoft.AspNetCore.Components.QuickGrid.EnableUrlBasedQuickGridNavigationAndSorting`
to `false` to restore button controls. Update CSS that assumes controls are
`<button>` because the default controls are now `<a>`.

## Culture and localization startup

Standalone WebAssembly loads globalization resources for
`CultureInfo.DefaultThreadCurrentUICulture` as well as
`DefaultThreadCurrentCulture` (10.0).

Prerendered Interactive WebAssembly components persist the server's
`CurrentCulture` and `CurrentUICulture` in component state and apply them
before client satellite assemblies load (11.0-preview.5). Disable this when
the client must independently choose culture:

```csharp
builder.Services.AddRazorComponents()
    .AddInteractiveWebAssemblyComponents(options =>
        options.UseCultureFromServer = false);
```

## Render-mode boundaries

Components with `@rendermode` can receive `ChildContent` and other non-generic
`RenderFragment` parameters from a different render mode
(11.0-preview.5). Blazor prerenders the fragment on the server and rehydrates
the captured render tree rather than trying to serialize a delegate.

```razor
<MyComponent @rendermode="InteractiveServer">
    <p>Server-rendered child content</p>
</MyComponent>
```
