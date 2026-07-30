# Rendering and labeling

## Rendering extent and temporal behavior

### Buffer symbol-rendering extents

Since 3.42, a symbol can request a configurable buffer around the canvas
extent. Features outside the visible extent are then considered when their
generated symbols may extend into view. Geometry generators such as
`buffer(@geometry, 7)` need enough buffer for correct edge rendering; larger
buffers increase the amount of data considered and therefore trade
performance for correctness.

### Accumulate temporal raster pixels

Since 4.0, raster layers in **represent temporal values** mode can accumulate
pixels over time. This matches the cumulative single-date/time behavior
already available for vector features, allowing raster and vector animation
frames to retain earlier values together.

### Calculate mesh color ranges from the view

Since 3.42, mesh color-ramp minimum and maximum values can be calculated from
the current canvas extent. The range may be locked to one canvas or follow the
active canvas, paralleling raster rendering behavior.

## Raster and vector labeling

### Label raster pixels

Since 3.42, raster layers can label pixels from a selected band through the
normal labeling engine. Raster labels participate in conflict handling and
support numeric formatting, text effects, priority, z-index, scale
visibility, pixel-size visibility, and optional values resampled over
neighboring pixels.

### Set custom tab stops

Since 3.42, label formatting accepts a list of tab-stop distances. Use it when
successive tabs need different offsets instead of one distance applied to
every tab.

### Style HTML label blocks and spans

Since 3.42, the text renderer supports `background-color` and
`background-image` on block and inline HTML. Block elements accept margins in
points, and `line-height` accepts points or percentages:

```html
<div style="margin: 5pt 0pt 10pt 0pt; background-color: #fff; line-height: 120%">Text</div>
```

HTML backgrounds do not work on curved text. Negative margins are limited to
the bottom margin.

### Separate labels across layers

Since 3.44, a vector label can reserve a margin that excludes labels from any
layer. A separate duplicate-prevention option suppresses matching label text
within a minimum distance across all layers, using case-sensitive comparison.
These controls solve different problems: proximity clearance versus repeated
text.

### Ignore whitespace in curved-label collisions

Since 4.0, curved label placement has a data-definable option to ignore spaces
and tabs during label-to-label and label-to-obstacle collision tests. The
option is off by default and unavailable for non-curved placement modes.

### Distribute multipart labels

Since 4.0, multipart labeling offers three behaviors: label only the largest
part, put the same text on every part, or split newline-delimited label lines
across parts. Splitting happens after the existing wrap-character setting.
When there are more lines than geometry parts, the surplus lines are not
rendered.

### Choose curved-label placement

Since 4.0, curved labels can place successive characters at line vertices,
stretch character spacing to fill the line, or stretch word spacing to fill
the line. The mode can be data-defined per feature. Vertex placement omits
characters that exceed the number of available vertices.

## Symbology and styles

### Edit blank templated-line segments

Since 4.0, the blank-segment map tool creates, selects, deletes, and resizes
per-feature gaps where a templated line omits hashes or markers. Segment data
is stored in a data-defined field or auxiliary-storage property, backed by
templated-line symbol-layer support.

### Edit additional templated-line items and trim ends

Since 4.2, hash and marker line symbol layers accept start and end trim
distances. Templated-line tools can also create, move, rotate, and delete
additional hashes or markers. Added items share the original item's style and
state.

### Reuse selective-masking presets

Since 4.0, mask sources can be saved as named presets and linked from multiple
layers. Changing a preset's source selection immediately updates every linked
layer. Select `custom` to retain the earlier per-layer configuration.

### Transfer named styles in bulk

Since 4.0, the layer-tree menu can copy and paste every named style between
layers in one operation. It also supplies grouped style-category shortcuts for
transferring related sets of style properties.

### Render and edit annotations

Since 4.0, an annotation selection tool can multi-select, move, delete, resize,
and rotate annotation items. Annotation layers can render marker and text
items as 3D billboards. Marker billboards support terrain clamping, offsets,
and callout lines; text billboards use a separately configurable 3D text
format.

## Scale and legend participation

### Select project-wide scale calculation

Since 3.44, projects can calculate scale at the map top, bottom, middle,
horizontal average, or equator. The choice affects new layout scale bars,
displayed and API values including `@map_scale`, scale-based visibility,
Processing map renders, and server renders. It does not affect symbol sizes in
map units. Equator mode is latitude-independent only for degree-based CRSs.

### Control automatic layout-legend inclusion

Since 4.0, vector, raster, mesh, and point-cloud layer properties include an
enabled-by-default setting that controls whether the layer is automatically
included in print-layout legends.
