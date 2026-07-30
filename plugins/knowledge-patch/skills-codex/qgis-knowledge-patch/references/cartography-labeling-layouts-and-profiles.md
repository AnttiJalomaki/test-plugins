# Cartography, Labeling, Layouts, and Profiles

## Symbols, labeling, and annotations

### Symbol rendering extent buffers (3.42)

Symbols can request a configurable buffer around the canvas extent so
off-canvas features are still considered when generated symbols extend into
view. This handles expressions such as `buffer(@geometry, 7)`; a larger buffer
improves correctness at a rendering-performance cost.

### Raster pixel labeling (3.42)

Raster layers can label pixels from a selected band through the normal
labeling engine. This includes conflict handling, numeric formatting, text
effects, priority, scale and pixel-size visibility, z-index, and optional
values resampled over neighboring pixels.

### Custom label tab stops (3.42)

Label formatting can use a list of custom tab-stop distances instead of one
distance for every tab.

### Expanded CSS for HTML labels (3.42)

The text renderer supports `background-color` and `background-image` on block
or inline HTML, point-unit margins on block elements, and `line-height` in
points or percent:

```html
<div style="margin: 5pt 0pt 10pt 0pt; background-color: #fff; line-height: 120%">Text</div>
```

Backgrounds do not work on curved text. Negative margins are limited to the
bottom margin.

### Cross-layer label separation (3.44)

Vector labels can reserve a margin that prevents labels from any layer being
placed nearby. A separate duplicate-prevention setting suppresses matching
case-sensitive text within a minimum distance across all layers.

### Editable blank segments in line symbology (4.0)

The blank-segment map tool creates, selects, deletes, and resizes per-feature
gaps where a templated line omits hashes or markers. Segments live in a
data-defined field or auxiliary-storage property, backed by templated-line
symbol layers.

### Editable templated-line items (4.2)

Hash and marker line symbol layers have start/end trim distances.
Templated-line tools can create, move, rotate, and delete additional hashes or
markers that share the original item's style and state.

### Annotation editing and 3D billboards (4.0)

A selection tool can multi-select, move, delete, resize, and rotate annotation
items. Annotation layers can render marker and text 3D billboards. Marker
billboards support terrain clamping, offsets, and callout lines; text
billboards use a separately configurable 3D text format.

### Whitespace-aware curved-label collisions (4.0)

Curved placement has a data-definable option to ignore spaces and tabs during
label/obstacle collision tests. It is off by default and is unavailable for
non-curved placement.

### Multipart label distribution (4.0)

Multipart labeling can use the largest part only, repeat the same text on
every part, or split newline-delimited label lines across parts. Splitting
occurs after the wrap-character setting. Surplus lines are not rendered when
there are too few parts.

### Curved-label placement modes (4.0)

Curved labels can place characters at successive line vertices, stretch
character spacing to line length, or stretch word spacing to line length. The
mode can be data-defined per feature. Vertex placement drops excess characters
when the line has too few vertices.

### Shared selective-masking presets (4.0)

Mask sources can be stored in named presets shared by many layers. Editing a
preset updates every linked layer immediately; `custom` retains per-layer
configuration.

### Automatic layout-legend inclusion (4.0)

Vector, raster, mesh, and point-cloud layer properties have an enabled-by-
default setting controlling automatic inclusion in print-layout legends.

### WMS highlight label frames (4.2)

QGIS Server WMS highlight labels accept frame styling through
`HIGHLIGHT_LABELFRAMEBACKGROUNDCOLOR`,
`HIGHLIGHT_LABELFRAMEOUTLINECOLOR`,
`HIGHLIGHT_LABELFRAMEOUTLINEWIDTH`, and
`HIGHLIGHT_LABELFRAMESIZE`. Parameters can be map-scoped, for example
`MAP0:HIGHLIGHT_LABELFRAMESIZE=5`.

## Layouts, legends, and charts

### Auto-wrapped layout legends (3.44)

Layout legend text can wrap automatically after a configured line length
measured in millimeters.

### Layout-legend synchronization modes (4.0)

All Project Layers, Visible Layers, and Manual Layer Selection replace the
Auto update checkbox; Reset replaces Update All. New legends default to
Visible Layers, which follows visibility and layer-tree changes but does not
filter by map extent. A global layout option can restore the previous default.

### Data-defined layout-grid annotations (4.0)

Each grid annotation can be shown or hidden by expression using `@grid_axis`,
`@grid_number`, `@grid_count`, and the one-based per-axis `@grid_index`.

### Atlas frame and coverage controls (4.0)

An atlas polygon can reshape a layout map item's frame for clipping and
masking. A separate option renders only the current coverage feature, avoiding
expressions that hide the other coverage features.

### Data-driven layout charts (4.0)

Print and atlas layouts can contain chart items whose X and Y series come from
source-layer expressions. Bar and line charts support filtering and ordered
iteration, and pie charts are available.

### Renderer-derived layout charts (4.2)

Layout charts can derive X-axis categories from a source vector layer's
renderer and reuse its symbol colors. A blank series counts matching features;
a Y expression can instead sum a field or calculated value.

### Shape-clipped layout pictures (4.2)

A layout picture can be clipped by a shape item. Both pictures and shapes may
be driven dynamically by atlas attributes.

### Layer-tree-aware Geospatial PDFs (4.2)

When a layout map item has no locked layers, Geospatial PDF export can preserve
project groups, nesting, order, names, visibility, and group layers. Visible
and invisible project layers are exported. Attributes are enabled for all
layers or none, and mutually exclusive groups are unsupported.

### Localized project and layer metadata (4.0)

Key project and layer metadata participates in project translation.
Translations can feed layout labels, map decorations, and other consumers.

## Expressions, scales, and profile presentation

### String and CRS expression functions (3.44)

Expressions include string forms of `repeat` and `reverse`, `crs_from_text`
for authority codes, WKT, or PROJ definitions, and `crs_to_authid` for an
`authority:id` result:

```qgis
repeat('ab', 3)
reverse('abc')
crs_to_authid(crs_from_text('EPSG:4326'))
```

### Magnetic-model expressions (4.0)

`magnetic_declination`, `magnetic_inclination`,
`magnetic_declination_rate_of_change`, and
`magnetic_inclination_rate_of_change` return angles or annual rates at a
point.

### Time-zone expressions (4.0)

`timezone_from_id`, `timezone_id`, and `get_timezone` create or inspect IANA
time zones. `convert_timezone` changes a datetime to the equivalent instant in
another zone; `set_timezone` replaces its zone without changing date or time
components.

### Cubic Bézier scaling and joined concatenation (4.2)

`scale_cubic_bezier` performs cubic Bézier interpolation and supports MapBox
`cubic-bezier` style conversion. `concat_ws(separator, ...)` ignores NULL
arguments:

```qgis
concat_ws('-', 'a', NULL, 'b')
```

This returns `a-b`.

### Project-wide scale calculation methods (3.44)

Projects can calculate scale at the map top, bottom, middle, horizontal
average, or equator. The setting affects new layout scale bars, displayed and
API values such as `@map_scale`, scale-based visibility, Processing map
renders, and server renders, but not symbol sizes in map units. Equator mode
is latitude-independent only for degree-based CRSs.

### Per-layer elevation-profile tolerance (3.42)

A vector layer's elevation properties can set `custom tolerance`, overriding
the elevation profile widget's global tolerance for that layer.

### Profile subsection indicators (3.44)

Elevation profiles and Print Layout profile elements can show subsection
indicators as vertical lines with custom symbology.

### Continuous point-cloud profile lines (4.0)

Profiles can render a point cloud as a continuous elevation line instead of
points; tolerance controls sparse results. A lockable distance:elevation scale
ratio can replace the normal 1:1 navigation ratio.

### Elevation profiles in 3D (4.2)

Show Profile in 3D Views displays a profile plot's elevation curve in 3D,
derives Z limits from data within the curve, and links cursor position through
line or polygon rubber bands.
