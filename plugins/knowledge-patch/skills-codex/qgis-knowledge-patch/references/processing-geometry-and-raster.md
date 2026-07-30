# Processing, Geometry, and Raster Workflows

## Processing outputs and workflow controls

### Layer-metadata Processing algorithms (3.42)

Native algorithms can copy metadata between layers, apply metadata from QMD,
export metadata to QMD, append history, update only non-empty fields, and set
basic fields including identifiers, title, language, CRS, abstract, and fees.

### Expression-driven scatterplot hover text (3.42)

Vector Layer Scatterplot can derive optional hover text from a QGIS
expression.

### Merge Vector Layers provenance fields (3.42)

Merge Vector Layers can add source `layer` and `path` attributes. The option is
enabled by default for backward compatibility.

### Plot titles and logarithmic axes (3.42)

Scatterplot, Barplot, and Boxplot accept plot and axis titles. An empty axis
title falls back to the field name; a single space suppresses it. Scatterplots
also support logarithmic scaling on either axis.

### Processing workflow additions (3.44)

QGIS includes a native clone of SAGA Fill Sinks (Wang & Liu), including the
source implementation's existing behavior and bugs. Both Raster Calculator
interfaces expose raster creation options. Batch Processing accepts temporary
output layers.

### Named temporary Processing outputs (4.0)

Processing results can remain temporary while carrying a user-selected layer
name. The memory-chip icon continues to identify temporary results.

### Hub-distance algorithm replacement (4.0)

The C++ Hub Distance algorithm replaces Distance to Nearest Hub (Points) and
Distance to Nearest Hub (Line to Hub) and provides both optional outputs. The
two older algorithms are deprecated.

## Geometry checking, digitizing, and vector output

### Geometry checks and fixes in Processing (3.42)

Geometry Checker operations appear under Check Geometry and Fix Geometry.
Checks output an error-only layer plus point error locations and identifiers.
Fixes output a corrected layer, point locations, and a per-feature fix report.

### Feature arrays along a line (4.0)

A map tool can copy point, line, or polygon features into an array distributed
along a line.

### Bézier, chamfer, and fillet digitizing (4.0)

The poly-Bézier/freeform tool creates NURBS curves by dragging anchors and
handles; `Alt`+click resets a point's handles. Polygon digitizing has chamfer
and fillet tools. CAD floaters can show Cartesian or ellipsoidal area and
total length/perimeter. Digitizing remains Cartesian, so ellipsoidal display
can differ.

### Polygon orientation algorithms (4.0)

`native:forcecw` produces clockwise exterior and counter-clockwise interior
rings; `native:forceccw` uses the inverse convention. `native:forcecw`
replicates the existing right-hand-rule operation.

### Network validation algorithms (4.0)

Validate Network reports invalid direction values, near-but-unconnected nodes,
and nodes too close to segments. It returns bad source features plus line
features describing topology errors. Extract Network Endpoints identifies
sources/sinks by edge direction or degree-one dead ends regardless of
direction.

### Z-aware reprojection (4.0)

Reproject Layer has an optional boolean parameter to transform Z coordinates
along with horizontal coordinates.

### Vector audit, filtering, and packaging outputs (4.0)

Delete Duplicate Geometries can emit removed duplicates. Remove Parts by
Length/Area drops undersized parts or entire single-part features. Package
Layers can transform to a destination CRS and filter every input by extent; a
packaged layer is still created empty when no features intersect.

### Native SFCGAL integration (4.0)

QGIS can use SFCGAL through `QgsSfcgalEngine` and the conversion-reducing
`QgsSfcgalGeometry` wrapper. Approximate Medial Axis makes a simplified line
skeleton from a shape's 2D projection and ignores Z.

### Geometry-check and medial-axis controls (4.2)

Check Holes has an area threshold that excludes holes larger than the
threshold from error results. With SFCGAL 2.3, Approximate Medial Axis accepts
`extendToEdges` to extend skeleton endpoints to the polygon boundary.

### Concave hulls of polygons (4.2)

Concave Hull by Feature accepts polygons and lines directly, so polygon
interiors are considered without vertex extraction. Fill Gaps Between
Polygons creates an outer, possibly non-convex hull without intersecting
polygon interiors. The GEOS concave-hull-of-polygons function is exposed to
PyQGIS.

## Raster and terrain processing

### Raster-extrema extraction (3.42)

Raster Zonal Min/Max emits minimum and maximum pixel-center points for every
polygon zone. Extract Min/Max Pixel emits extrema for a selected raster band
and returns only one point when multiple pixels tie.

### Elevation-profile image generation (3.42)

A Processing algorithm renders elevation-profile images, allowing a model to
generate profiles for multiple curves.

### Raster rank (3.44)

Raster rank selects a requested rank from input raster values per cell. For
`[10, 20, 30, 40]`, ranks `2` and `-2` return `20` and `30`. By default,
NoData is excluded and only an unavailable rank yields NoData. An alternate
mode propagates NoData if any input cell is NoData.

### Explicit Cloud Optimized GeoTIFF output (4.0)

Raster export and Save dialogs can request COG optimization and pyramids when
GDAL 3.13+ is available. A Processing algorithm can bulk-convert a raster
directory. Processing must pass `-of COG` explicitly because COG and GTiff
share `.tif` and `.tiff` extensions.

### Terrain-raster processing controls (4.0)

Processing includes feature-preserving DEM smoothing, native Gaussian blur,
and total-curvature algorithms. Slope, aspect, hillshade, and ruggedness expose
output NoData and raster-creation options.

### GDAL dataset identification (4.0)

GDAL Data Identification exposes automated dataset metadata extraction and
requires GDAL 3.13 or later.

### WMS-aware raster extraction (4.0)

Clip Raster by Extent and Clip Raster by Mask Layer can request WMS input at a
reference scale and service resolution, preserving scale-dependent rendering.
Service resolution defaults to 96 DPI. Supporting APIs are
`QgsProcessingRasterLayerDefinition` and `QgsWmsUtils`.

## Temporal and coordinate behavior

### Accumulating temporal raster pixels (4.0)

Raster layers in represent-temporal-values mode can accumulate pixels over
time, matching cumulative single-date/time vector behavior so raster and
vector animation frames advance together.

### Topocentric CRS support (4.2)

QGIS supports topocentric coordinate reference systems. An origin-point widget
is enabled when Topocentric CRS is explicitly selected in the CRS chooser.
