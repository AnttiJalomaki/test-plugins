# iOS and Darwin

## Sources, style loading, and rendering

### PMTiles

MapLibre Native iOS 6.10 supports PMTiles sources through the `pmtiles://` URL
scheme. From 6.14, PMTiles metadata is always treated as using the XYZ tile
scheme (`ios-6.10.0`).

### Style JSON initialization and cancellation

From 6.13, `MLNMapView` can be initialized with style JSON
(`ios-6.10.0`). In 6.22, loading style JSON cancels any pending style request,
so the JSON load supersedes the request already in flight
(`ios-6.20.0`).

### MLT and synchronous GeoJSON

iOS 6.20 parses vector-tile sources in MLT format (`ios-6.20.0`). Version
6.22 adds synchronous GeoJSON source updates for changes that must be applied
before execution continues.

### Vector-tile feature state

From iOS 6.15, feature-state updates in `GeometryTile` and
`SourceFeatureState` are applied completely for vector-tile layers. Dynamic
feature-state code does not need to compensate for the earlier partial-update
behavior (`ios-6.15.0`).

### Color relief and texture atlases

iOS 6.24 adds Color-Relief Layer support and updates the hillshade algorithms
(`ios-6.20.0`).

iOS 6.14 introduced a dynamic texture atlas. When adopting it, verify that
glyphs still load correctly in existing styles (`ios-6.10.0`).

## Camera, controls, and snapshots

iOS supports constraining the camera to maximum bounds
(`ios-6.10.0`). Version 6.20 makes the rotate gesture's threshold for
snapping back to north configurable (`ios-6.20.0`).

Version 6.21 adds frustum offset, allowing rendering to omit screen edges, and
also permits attribution to be hidden. The same release supports adding extra
annotations to snapshotter output (`ios-6.20.0`).

iOS 6.23 adds basic camera-roll support (`ios-6.20.0`).

Version 6.25 exports `MLNScaleBar`, allowing application code to reference the
SDK scale-bar type directly (`ios-6.25.0`).

## Custom rendering and diagnostics

### Custom layers

iOS 6.11 supports custom style layers defined from Swift and adds an iOS
method to trigger repainting (`ios-6.10.0`). Version 6.12 introduces custom
drawable layer v3.

Version 6.28 adds `nearClippedProjectionMatrix` to `MLNCustomStyleLayer`,
giving custom style layers the near-clipped projection matrix
(`ios-6.25.0`).

### Headless Metal and action journal

Version 6.26.1 exposes the Metal texture from the headless backend for direct
Metal interoperability (`ios-6.25.0`).

iOS 6.15 adds action-journal support and an example showing its adoption
(`ios-6.15.0`).

### Rendering statistics

iOS includes a rendering-statistics view for inspecting renderer activity in
an application (`ios-6.15.0`).

## Observers and networking

iOS and macOS observer hooks arrive in 6.12, and 6.13 adds the previously
missing `sourceDidChange` event (`ios-6.10.0`).

In 6.28, layer observers are notified when a layer's source layer or source ID
changes (`ios-6.25.0`).

iOS 6.20 adds network delegate methods, expanding the networking lifecycle
visible to delegates (`ios-6.20.0`). In 6.28,
`MLNNetworkConfiguration` forwards `didReceiveResponse` to its delegate, so a
delegate can rely on receiving the response callback (`ios-6.25.0`).

Version 6.14 adds range-request support to `AssetFileSource`
(`ios-6.10.0`).

## Attribution and offline identifiers

From iOS 6.15, `MLNSource.attributionHtmlString` exposes a source's HTML
attribution for custom attribution interfaces (`ios-6.15.0`).

Starting in 6.25, `MLNOfflinePack` exposes its underlying offline region
identifier on Darwin and iOS (`ios-6.25.0`).

## Distribution

Starting with 6.21.1, releases include a static XCFramework
(`ios-6.20.0`).

The Swift package comes from the distribution-only repository below, while
issues belong in the main `maplibre-native` repository. Import the resulting
module as `MapLibre` (`ios-sdk`):

```text
https://github.com/maplibre/maplibre-gl-native-distribution
```

## Cocoa style APIs

### Runtime-style vocabulary

The iOS API renames several style-spec concepts (`ios-sdk`):

| Style-spec term | Cocoa API term |
| --- | --- |
| `bounds` | coordinate bounds |
| `filter` | predicate |
| function type | interpolation mode |
| `id` | identifier |
| `image` | style image |
| `layer` | style layer |
| `property` | attribute |
| SDF icon | template image |
| `source` | content source |

### Expressions and predicates

Layout and paint attributes take `NSExpression` values backed by Cocoa types
such as `UIColor`, `CGVector`, and `UIEdgeInsets`; the older style-function API
is unsupported (`ios-sdk`).

Set `MLNVectorStyleLayer.predicate` with `NSPredicate`. Translate style
operators to predicate constructs such as `key != nil`, `key IN {…}`, `AND`,
`OR`, and `NOT`.

### Typed property names

Several native property names are not direct transforms of their JSON names
(`ios-sdk`):

| Style property | Typed iOS property |
| --- | --- |
| `line-dasharray` | `lineDashPattern` |
| `raster-hue-rotate` | `rasterHueRotation` |
| `icon-image` | `iconImageName` |
| `icon-size` | `iconScale` |
| `text-field` | `text` |
| `text-font` | `textFontNames` |
| `text-size` | `textFontSize` |

Formatted symbol text uses `.fontNamesAttribute`, `.fontScaleAttribute`, and
`.fontColorAttribute`.

### Coordinate ordering

`MLNCoordinateQuad` lists image-source corners counterclockwise, opposite the
style specification's clockwise order. `UIEdgeInsets(top:left:bottom:right:)`
is likewise counterclockwise while style-spec padding is clockwise
(`ios-sdk`).

## Imported offline databases

`MLNOfflineStorage.sharedOfflineStorage.packs` is the canonical in-memory pack
collection. After importing an offline database by file or URL, call
`reloadPacks` before using that collection so it reflects the merged regions
(`ios-sdk`).

## Metal plugin layers

Darwin plugin layers are Metal-only and differ from ordinary annotations and
GL JS custom-layer interfaces (`ios-sdk`). Subclass `MLNPluginLayer`, register
the class with the map view, and implement the class method
`layerCapabilities` so the style parser can instantiate the declared layer
type.

Capabilities declare the layer ID, render-pass requirements, and typed paint
properties with defaults. Put initialization values in the style layer's
`properties` object and expression-capable values in `paint`.

```objc
+ (MLNPluginLayerCapabilities *)layerCapabilities {
    MLNPluginLayerCapabilities *caps = [MLNPluginLayerCapabilities new];
    caps.layerID = @"plugin-layer-metal-rendering";
    caps.requiresPass3D = YES;
    caps.layerProperties = @[
        [MLNPluginLayerProperty propertyWithName:@"scale"
                                    propertyType:MLNPluginLayerPropertyTypeSingleFloat
                                    defaultValue:@1.0]
    ];
    return caps;
}
```
