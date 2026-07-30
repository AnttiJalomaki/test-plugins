# iOS and Darwin SDK

## Distribution and initialization

### Swift Package repository

The Swift package is published from the distribution-only repository below and
is imported as `MapLibre`. File issues in the main `maplibre-native`
repository.

```text
https://github.com/maplibre/maplibre-gl-native-distribution
```

Starting with 6.21.1, GitHub releases also include a static XCFramework.

### Style JSON initialization and cancellation

Within the `ios-6.10.0` extraction line, iOS 6.13 allows `MLNMapView` to be
initialized with style JSON. From 6.22, loading style JSON cancels any pending
style request, so the JSON load supersedes the in-flight request.

### Asset range requests and texture atlas

iOS 6.14 adds range-request support to `AssetFileSource`. It also introduces a
dynamic texture atlas; after adopting it, verify that glyphs still load
correctly for existing styles.

## Sources and dynamic state

### PMTiles

iOS 6.10 adds PMTiles sources through the `pmtiles://` URL scheme. From 6.14,
PMTiles metadata is always treated as using the XYZ tile scheme.

### MLT and GeoJSON

At the `ios-6.20.0` boundary, iOS can parse vector-tile sources in MLT format.
iOS 6.22 adds synchronous GeoJSON updates for changes that must be applied
before execution continues.

### Complete vector-tile feature state

At `ios-6.15.0`, feature-state updates in `GeometryTile` and
`SourceFeatureState` are fully applied for vector-tile layers. Remove
workarounds for the earlier incomplete-update behavior.

### Attribution HTML

`MLNSource.attributionHtmlString` is exposed from iOS 6.15, allowing a custom
attribution interface to retrieve the source's HTML attribution directly.

## Camera and map controls

iOS can constrain the camera to maximum bounds. From 6.20, the rotate gesture's
threshold for snapping the map back to north is configurable.

iOS 6.21 adds a frustum offset so the renderer can omit screen edges. iOS 6.23
adds basic camera-roll support.

Version 6.25, represented by the `ios-6.25.0` extraction boundary, publicly
exports `MLNScaleBar`, allowing application code to refer directly to the SDK
scale-bar type.

## Snapshotting and attribution

iOS 6.21 allows attribution to be hidden and supports extra annotations in
snapshotter output.

## Layers and renderer inspection

### Custom layers

iOS 6.11 supports defining custom style layers from Swift and provides a method
to trigger repainting. iOS 6.12 introduces custom drawable layer v3.

In 6.28, `MLNCustomStyleLayer` gains `nearClippedProjectionMatrix`, exposing the
near-clipped projection matrix to custom style layers.

### Color relief and hillshade

iOS 6.24 adds Color-Relief Layer support and updates the hillshade algorithms.

### Rendering statistics

iOS 6.15 provides a rendering-statistics view for inspecting renderer activity
inside an application.

### Headless Metal interoperability

iOS 6.26.1 exposes the Metal texture from the headless backend for consumers
that need direct Metal interoperability.

## Observers, journaling, and networking

### Observer hooks

iOS and macOS observer hooks arrive in 6.12. Version 6.13 adds the previously
missing `sourceDidChange` event. From 6.28, layer observers are notified when a
layer's source layer or source ID changes.

### Action journal

iOS 6.15 adds action-journal support and ships an iOS adoption example.

### Network delegates

iOS 6.20 expands network delegate methods and the available networking
lifecycle. In 6.28, `MLNNetworkConfiguration` forwards `didReceiveResponse` to
its delegate, so delegate implementations can rely on receiving that callback.

## Cocoa style vocabulary

The iOS API uses names that differ from style-spec terminology:

| Style-spec term | Cocoa API term |
| --- | --- |
| bounds | coordinate bounds |
| filter | predicate |
| function type | interpolation mode |
| id | identifier |
| image | style image |
| layer | style layer |
| property | attribute |
| SDF icon | template image |
| source | content source |

## Foundation expressions and predicates

Layout and paint attributes take `NSExpression` values backed by Cocoa types
such as `UIColor`, `CGVector`, and `UIEdgeInsets`. The older style-function API
is unsupported.

Set `MLNVectorStyleLayer.predicate` with `NSPredicate`. Translate style
operators to predicate syntax such as `key != nil`, `key IN {…}`, `AND`, `OR`,
and `NOT`.

Typed property names are not always mechanical conversions:

- `line-dasharray` becomes `lineDashPattern`.
- `raster-hue-rotate` becomes `rasterHueRotation`.
- `icon-image` and `icon-size` become `iconImageName` and `iconScale`.
- `text-field`, `text-font`, and `text-size` become `text`, `textFontNames`,
  and `textFontSize`.
- Formatted text uses `.fontNamesAttribute`, `.fontScaleAttribute`, and
  `.fontColorAttribute`.

## Coordinate ordering

`MLNCoordinateQuad` lists image-source corners counterclockwise, opposite the
style specification's clockwise order. `UIEdgeInsets(top:left:bottom:right:)`
is also counterclockwise, while style-spec padding is clockwise.

## Offline packs

`MLNOfflineStorage.sharedOfflineStorage.packs` is the canonical in-memory pack
collection. After importing an offline database by file or URL, call
`reloadPacks` before reading the collection so it reflects merged regions.

From iOS 6.25, `MLNOfflinePack` exposes its underlying offline region
identifier on Darwin and iOS.

## Metal plugin layers

Darwin plugin layers are Metal-only and differ from ordinary annotations and
browser custom-layer interfaces. Subclass `MLNPluginLayer`, register the class
with the map view, and implement `layerCapabilities` so the style parser can
instantiate the declared layer type.

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
