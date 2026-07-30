# Processing and analysis

## Metadata, provenance, and workflow outputs

### Manage layer metadata

Since 3.42, the native toolbox includes algorithms to copy metadata between
layers, apply metadata from QMD, export metadata to QMD, append history, update
only non-empty fields, and set basic fields. The basic fields include
identifiers, title, language, CRS, abstract, and fees.

### Preserve merge provenance

Since 3.42, **Merge Vector Layers** can add source `layer` and `path`
attributes to the output. The option is enabled by default for backward
compatibility.

### Name temporary outputs

Since 4.0, Processing outputs can remain temporary while carrying a
user-selected layer name. The memory-chip icon continues to identify these
layers as temporary. Since 3.44, Batch Processing also accepts temporary
output layers.

## Geometry checks, repair, and construction

### Check and fix geometry

Since 3.42, Geometry Checker operations appear in the Processing Toolbox under
**Check Geometry** and **Fix Geometry**. Checks output an error-only layer plus
point error locations and identifiers. Fixes output the corrected layer,
point locations, and a per-feature fix report.

Since 4.2, **Check Holes** has an area threshold that excludes holes larger
than the threshold from error results.

### Force polygon orientation

Since 4.0, `native:forcecw` produces clockwise exterior rings and
counter-clockwise interior rings. `native:forceccw` applies the opposite
convention. `native:forcecw` duplicates the behavior of the existing
right-hand-rule operation.

### Use SFCGAL geometry operations

Since 4.0, QGIS can use SFCGAL through `QgsSfcgalEngine` and the
conversion-reducing `QgsSfcgalGeometry` wrapper. **Approximate Medial Axis**
creates a simplified line skeleton from a geometry's 2D projection and ignores
Z. Since 4.2, SFCGAL 2.3 enables its `extendToEdges` option, extending skeleton
endpoints to the polygon boundary.

### Build concave polygon hulls and fill gaps

Since 4.2, **Concave Hull by Feature** accepts polygon and line inputs
directly, so polygon interiors participate without prior vertex extraction.
**Fill Gaps Between Polygons** creates an outer, potentially non-convex hull
without intersecting polygon interiors. The underlying GEOS
concave-hull-of-polygons operation is also exposed to PyQGIS.

## Raster calculations and extraction

### Extract raster extrema

Since 3.42, **Raster Zonal Min/Max** emits minimum- and maximum-value
pixel-center points for every polygon zone. **Extract Min/Max Pixel** emits
extrema for a chosen raster band and returns only one point when several
pixels tie for an extreme.

### Calculate raster rank

Since 3.44, **Raster rank** chooses a requested rank from the input raster
values at each cell. For `[10, 20, 30, 40]`, ranks `2` and `-2` return `20`
and `30`. By default it excludes NoData and produces NoData only when the rank
is unavailable. Its alternate mode propagates NoData whenever any input cell
is NoData.

### Fill sinks and control raster creation

Since 3.44, the native toolbox includes a clone of SAGA's **Fill Sinks
(Wang & Liu)**, including the source implementation's existing behavior and
bugs. Raster-creation options are exposed in both Raster Calculator
interfaces.

### Smooth and analyze terrain rasters

Since 4.0, Processing includes feature-preserving DEM smoothing, native
Gaussian blur, and total-curvature algorithms. Slope, aspect, hillshade, and
ruggedness expose output NoData and raster-creation options.

### Export explicit Cloud Optimized GeoTIFFs

Since 4.0, raster export and Save dialogs can request COG optimization and
pyramids with GDAL 3.13+. A Processing algorithm can bulk-convert a raster
directory. Processing outputs must pass `-of COG` explicitly to distinguish
the COG and GTiff drivers, since both use `.tif` or `.tiff`.

### Extract rendered WMS rasters

Since 4.0, **Clip Raster by Extent** and **Clip Raster by Mask Layer** can
request a WMS input at a reference scale and service resolution, preserving
scale-dependent rendering. Service resolution defaults to 96 DPI. The
supporting APIs are `QgsProcessingRasterLayerDefinition` and `QgsWmsUtils`.

### Identify GDAL datasets

Since 4.0, **GDAL Data Identification** exposes automated dataset metadata
extraction and requires GDAL 3.13 or later.

## Vector auditing, transformation, and packaging

### Transform Z during reprojection

Since 4.0, **Reproject Layer** has an optional Boolean parameter that
transforms Z coordinates along with horizontal coordinates.

### Audit duplicates and remove small parts

Since 4.0, **Delete Duplicate Geometries** can emit the removed duplicates.
**Remove Parts by Length/Area** removes undersized parts or the entire feature
when a single-part feature is undersized.

### Filter and reproject packaged layers

Since 4.0, **Package Layers** can transform inputs to a destination CRS and
filter every input by an extent. When no feature intersects the extent, the
corresponding packaged layer is still created empty.

## Network analysis

### Validate a network

Since 4.0, **Validate Network** reports invalid direction values,
near-but-unconnected nodes, and nodes too close to segments. It returns bad
source features plus line features describing topology errors.

### Extract network endpoints

Since 4.0, **Extract Network Endpoints** identifies sources and sinks from edge
direction, or degree-one dead ends without regard to direction.

### Replace deprecated hub-distance algorithms

Since 4.0, the C++ **Hub Distance** algorithm replaces **Distance to Nearest
Hub (Points)** and **Distance to Nearest Hub (Line to Hub)**. Both older
algorithms are deprecated. The replacement supplies both optional outputs.

## Plots and profile products

### Add plot labels and logarithmic axes

Since 3.42, Scatterplot, Barplot, and Boxplot accept plot and axis titles. An
empty axis title falls back to the field name; a single space suppresses the
title. Scatterplots also support logarithmic scaling on either axis.

### Set scatterplot hover text

Since 3.42, **Vector Layer Scatterplot** can derive optional hover text from a
QGIS expression.

### Render elevation-profile images

Since 3.42, a Processing algorithm renders elevation-profile images and can
generate profiles for multiple curves from models.
