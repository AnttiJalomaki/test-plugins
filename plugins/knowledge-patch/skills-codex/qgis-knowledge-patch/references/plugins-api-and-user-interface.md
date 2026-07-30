# Plugins, APIs, and User Interface

## Plugin compatibility and application integration

### Advertise QGIS 4 compatibility with a version range

Plugin compatibility is derived from `qgisMinimumVersion` and the optional
`qgisMaximumVersion`. Without a maximum, support is assumed only through the
end of the minimum version's major line. To retain QGIS 3.22 support and join
the QGIS 4 Ready list, use:

```ini
[general]
qgisMinimumVersion=3.22
qgisMaximumVersion=4.99
```

The Ready list includes a plugin when either bound is at least 4.0.

### Remove the obsolete Qt 6 flag

`supportsQt6=True` is removed from core and is not recognized as a QGIS 4
compatibility declaration. Remove it and use the version range.

### Check the Qt 6 migration before widening compatibility

QGIS 4 plugins must replace Qt 5-only APIs and direct `PyQt5` imports with Qt 6
equivalents, preferably through `qgis.PyQt`, and must be tested on QGIS 4
before their metadata range is widened. Repository uploads run
`pyqgis4-checker`; its Qt6 Check tab identifies affected files and lines, but
findings do not block upload or approval. These three compatibility items form
the `qgis4-plugin-migration` guidance.

### Isolated QGIS 4 settings (4.2)

QGIS 4 stores settings separately from QGIS 3. On first startup, it makes a
one-time, lossless copy of the loaded QGIS 3 user profile. Later changes do not
synchronize, so installation, profile-management, and enterprise-deployment
scripts must target the QGIS 4 location.

### Plugin-delivered application themes (4.0)

Plugins can ship themes and custom application styles. Installing a plugin can
therefore alter the application theme without a corresponding core theme.

### User-defined menus and toolbars (4.0)

Users can create their own menus and toolbars instead of only customizing
built-in ones.

### Processing actions in custom UI (4.2)

A Processing algorithm can be assigned to a user-defined menu or toolbar.
Triggering the action opens the algorithm's parameter and execution dialog.

## Security, forms, and layer interaction

### Project trust for embedded Python (4.0)

Projects carry granular trust for macros, expression functions, actions, and
attribute-form initialization code. A trust dialog can preview code, while
global policy can allow or deny execution by project or path.

### Per-field remembered form values (4.0)

Attribute forms show a pin indicating whether the last captured value will be
reused, and users can toggle it. Layer form configuration controls session
reuse policies and their default, or can disable reuse for every field.

### Value-relation sorting (3.42)

Value Relation widgets can reverse their order or sort choices by a specified
field.

### Per-field merge policies (3.44)

Field configuration can choose the initial value used by Merge Features.
Policies include numeric Sum, Minimum, Maximum, and Geometry Weighted values;
Default Value; Unset Field, which falls back to the provider default or first
feature; Largest Geometry, based on length, area, or part count; and Set to
Null.

### SQL from the layer context menu (3.42)

Supported layers can execute SQL directly from their context menu in the
project table of contents.

### Bulk layer-style transfer (4.0)

The layer-tree menu can copy and paste every named style between layers in one
operation. Grouped style-category shortcuts transfer related sets of style
properties.

### Raw attribute copying (4.0)

Attribute tables and Identify Results can copy the literal provider value
instead of its represented value from locale formatting, expressions, or
display relations.

## PyQGIS and C++ APIs

### `QgsGeos` in PyQGIS (3.42)

PyQGIS exposes `QgsGeos` directly. Use it for GEOS-specific functionality that
is not available through the base `QgsGeometryEngine`.

### Dimensional `QgsGeometry.as_numpy()` output (3.42)

`QgsGeometry.as_numpy()` preserves dimensionality. Z and/or M geometries
return XYZ, XYM, or XYZM coordinates instead of always returning XY.

### Geometry and GeoPandas APIs (4.0)

`QgsGeometry.area3D()` returns surface area for polygons, polyhedral surfaces,
TINs, and collections, and zero for points and lines.
`QgsGeometryUtilsBase::pointsAreCollinear` handles 2D and 3D points, with
`QgsGeometryUtilsBase::points3DAreCollinear` available explicitly for 3D.
When GeoPandas is installed, `QgsVectorLayer.as_geopandas()` converts a layer
and its attributes to a GeoPandas dataframe.

### PyQGIS 3D extension points (4.0)

Plugins can derive custom canvas tools from `Qgs3DMapTool`, apply the
cross-section tool's four clipping planes, and call
`Qgs3DMapCanvas.castRay()` to obtain and manage 3D hits through `QgsRay3D`.

### GPS controls for plugins (3.44)

PyQGIS exposes `QgsAppGpsTools` through `iface.gpsTools()`. A plugin can create
a feature from the current track:

```python
iface.gpsTools().createFeatureFromGpsTrack()
```

It can also change the track symbol and update its geometry:

```python
iface.gpsTools().setGpsTrackLineSymbol(line_symbol)
```

### Persistent and synchronized elevation profiles (4.0)

Elevation profiles can be saved in the project, reopened, renamed, or removed
through the project-level profile manager. Opt-in Synchronize Layers to
Project mode mirrors the main layer tree, including groups and order, into a
profile. `QgsLayerTreeCustomNode` lets APIs represent non-layer application
objects in layer trees.
