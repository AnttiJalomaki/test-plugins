# Layouts, editing, and profiles

## Layout legends and export

### Wrap legend text

Since 3.44, layout legend text can wrap automatically after a configured line
length measured in millimeters.

### Select legend synchronization

Since 4.0, the layout legend's former Auto update checkbox is replaced by
**All Project Layers**, **Visible Layers**, and **Manual Layer Selection**.
**Reset** replaces **Update All**. New legends default to Visible Layers,
which follows visibility and layer-tree changes but does not filter by map
extent. A global layout option can restore the previous default.

### Export layer-tree-aware Geospatial PDFs

Since 4.2, Print Layout Geospatial PDF export can preserve project groups,
nesting, order, names, visibility, and group layers when the map item has no
locked layers. It exports both visible and invisible project layers.
Attributes are enabled for either all layers or none. Mutually exclusive
groups are not supported.

## Layout grids, atlas, pictures, and charts

### Control grid annotations by expression

Since 4.0, each layout-grid annotation can be shown or hidden by expression.
Available variables are `@grid_axis`, `@grid_number`, `@grid_count`, and the
one-based per-axis `@grid_index`.

### Reshape atlas frames and isolate coverage

Since 4.0, an atlas polygon can reshape a layout map item's frame for clipping
and masking. A separate atlas option renders only the current coverage
feature, avoiding expressions that hide all other coverage features.

### Build data-driven charts

Since 4.0, print and atlas layouts can contain chart items whose X and Y
series come from source-layer expressions. Bar and line charts support
filtering and ordered iteration; pie charts are also available.

Since 4.2, charts can derive X-axis categories from a source vector layer's
renderer and reuse renderer symbol colors. A blank series counts matching
features; a Y expression can instead sum a field or calculated value.

### Clip pictures by shapes

Since 4.2, a layout picture can be clipped by a shape item. Both the picture
and the clipping shape may be driven dynamically by atlas attributes.

## Feature and geometry editing

### Create feature arrays along a line

Since 4.0, a map tool can copy point, line, or polygon features into an array
distributed along a line.

### Digitize Bézier curves, chamfers, and fillets

Since 4.0, the poly-Bézier/freeform map tool creates NURBS curves by dragging
anchors and handles; `Alt`+click resets a point's handles. Polygon digitizing
also supports chamfer and fillet tools.

CAD floaters can display Cartesian or ellipsoidal area and total
length/perimeter. Digitizing remains Cartesian, so the ellipsoidal display may
differ from the resulting geometry.

### Snap georeferencer points

Since 3.42, the Georeferencer provides snapping options and the Advanced
Digitizing panel for placing reference points against existing geometry.

## Attribute forms and merge behavior

### Sort value relations

Since 3.42, Value Relation widgets can reverse their order or sort choices by
a specified field.

### Set per-field merge policies

Since 3.44, field configuration can define the initial value used by **Merge
Features**. Available policies are numeric Sum, Minimum, Maximum, and Geometry
Weighted values; Default Value; Unset Field; Largest Geometry; and Set to
Null.

Unset Field falls back to the provider default or the first feature. Largest
Geometry compares length, area, or part count as appropriate.

### Copy raw provider attributes

Since 4.0, attribute tables and Identify Results can copy the literal provider
value rather than the represented value produced by locale formatting,
expressions, or display relations.

### Remember form values per field

Since 4.0, attribute forms display a pin showing whether the last captured
value will be reused and allow the user to toggle it. Layer form configuration
can set session reuse policies and their default, or disable reuse for all
fields.

## Elevation profiles

### Override profile tolerance per layer

Since 3.42, a vector layer's elevation properties can set **custom tolerance**
to override the elevation profile widget's global tolerance for that layer.

### Mark profile subsections

Since 3.44, elevation profiles and Print Layout profile elements can display
subsection indicators as vertical lines with custom symbology.

### Save and synchronize profiles

Since 4.0, elevation profiles can be stored in the project and later reopened,
renamed, or removed from the project-level profile manager. Opt-in
**Synchronize Layers to Project** mirrors the main layer tree, including
groups and order, into a profile.

`QgsLayerTreeCustomNode` allows APIs to represent non-layer application
objects in layer trees.

## Coordinate reference systems

### Configure a topocentric CRS

Since 4.2, QGIS supports topocentric coordinate reference systems. An
origin-point widget is enabled when **Topocentric CRS** is explicitly selected
in the CRS chooser.
