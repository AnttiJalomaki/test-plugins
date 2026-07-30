# 3D, Mesh, and Point-Cloud Workflows

## 3D scenes and rendering

### 3D cross sections (3.44)

The cross-section tool collects start, end, and thickness points from the 2D
canvas, filters the 3D view to that possibly rotated region, and moves the
camera to a side view. Enabling or disabling the section does not reload the
whole scene.

### Globe scenes (3.44)

Globe-mode 3D scenes use a mesh following the project ellipsoid. Any map layer
can provide the 2D texture, and tiled-scene and point-cloud 3D renderers are
supported. A suitable CRS and ellipsoid can model another celestial body.

### Expanded 3D scene controls (4.0)

Cross sections can use a fixed editable width and be nudged left or right.
Scene export can omit terrain, extruded polygons can include floors, and a 3D
view can show a camera-centered 2D map overlay with an optional camera
frustum.

### Esri I3S scene layers (4.0)

The tiled-scene provider opens I3S 1.7+ `3DObject` and `IntegratedMesh` data
from ArcGIS REST or local SLPK files in 2D and 3D. It supports global
EPSG:4326 datasets and local projected-CRS datasets.

### Categorized and rule-based 3D rendering (4.2)

Vector 3D symbology supports categorized and rule-based renderers with
controls modeled on their 2D counterparts.

### Physically based 3D materials (4.2)

The physically based material supports base-color, metalness, roughness, and
ambient-occlusion texture maps. Metal-rough materials also support opacity,
solid emission color and strength, and data-defined base/emission colors.
Metal-rough and Phong textures expose data-defined scale, rotation, and
offset. Materials can be saved as tagged or favorited style-database presets.

### Configurable 3D model axes (4.2)

3D point-model symbols can select explicit up and forward axes instead of
assuming Z-up/Y-forward. This avoids manual rotations that interfere with
reusable rotation, scale, and data-defined settings.

### 3D environmental lighting and effects (4.2)

A cube-map skybox can generate environmental lighting dynamically for
physically based materials. This is optional and does not apply to fixed-
gradient backgrounds. 3D Effects also provides tone mapping, exposure, gamma,
light bloom, global MSAA, and configurable gradient backgrounds.

### Expanded 3D Tiles support (4.2)

QGIS renders 3D Tiles 1.0 `i3dm` and 1.1 glTF GPU instancing in both 2D and 3D,
including projected-CRS rotation correction. Quantized positions, oct-encoded
rotations, and feature IDs are unsupported. It also reads 1.1 implicit
quadtree tiling and 1.0 composite `cmpt` tiles.

### Coordinate-based 3D camera controls (4.2)

The camera dialog accepts target XYZ in map-CRS coordinates plus pitch,
heading, and distance. Optional live update pushes edits to the view, while
displayed values always follow camera movement. Vertical-axis inversion is
configured independently for walk dragging, captured-mouse walk mode, and
terrain mode.

### STL scene export (4.2)

3D scenes can be exported as STL as well as OBJ. STL is simpler and does not
preserve textures.

## Meshes

### View-dependent mesh color ranges (3.42)

Mesh color-ramp minimum and maximum values can be calculated from the current
canvas extent. They may be locked to one canvas or follow the active canvas,
like raster rendering.

### Mesh editing and elevation controls (3.42)

Adding a vertex can Delaunay-refine adjacent triangles by flipping
nonconforming edges. Actions can select all vertices or only isolated
vertices. New-vertex Z can prefer the mesh then a Z widget or terrain, or
always use project terrain or the Z widget. Selected vertices can infer Z from
project terrain.

### Mesh dataset-group management (3.42)

Externally added dataset groups may share names and are numbered
automatically. Added groups can be removed; groups belonging to the original
mesh source cannot.

### Mesh surface export (3.42)

Mesh: Surface to Polygon exports a mesh surface as a MultiPolygon layer.

### Mesh GetFeatureInfo responses (4.0)

QGIS Server can answer GetFeatureInfo requests for mesh layers.

## Virtual point clouds and COPC

### Virtual point-cloud overview behavior (3.42)

A VPC renders an overview when available and otherwise renders extents while
zoomed out. Styling can force extents only, overview only, or both.

### COPC Processing output (3.44)

PDAL Processing algorithms can write Cloud Optimized Point Cloud outputs
directly.

### Virtual point-cloud conversion, access, and editing (4.0)

Build VPC can convert LAS/LAZ inputs to COPC so the result is fully renderable.
VPC styling controls how early actual points replace extents or overviews.
Remote VPCs open directly, but editing requires the VPC and every linked COPC
file to be local.

### Multi-overview and zipped VPC (4.2)

Build VPC accepts `--overview-length`. Readers recognize every asset with the
`overview` role regardless of ID and render multiple overviews when zoomed
out. `QgsPointCloudLayer.overviews()` and
`QgsVirtualPointCloudProvider.overviews()` return lists. Zipped VPC datasets
open from `.vpz`.

## Point-cloud editing, analysis, and styling

### Point-cloud editing in 3D (3.44)

Select a point-cloud attribute and target value, then edit in 3D by polygon,
paintbrush, or above/below-line selection. An expression can limit which
selected points change.

### M3C2 point-cloud comparison (4.0)

Compare Point Clouds computes signed multiscale distances along locally
estimated surface normals through PDAL `filters.m3c2`. It is available only
when the QGIS build ships with PDAL later than 2.10.

### Point-cloud normalization and cleanup (4.0)

Height Above Ground adds a `HeightAboveGround` attribute or replaces Z using
nearby or triangulated ground points classified as class 2. Processing also
provides SMRF ground classification, statistical and radius noise filters,
and point-cloud translation, rotation, and scaling.

### Point-cloud TIN edge limits (4.0)

PDAL Export to Raster (TIN) can omit triangles whose edges exceed a maximum
length. This requires PDAL 2.6+ and wrench 1.2.2+.

### Per-layer point-cloud elevation shading (4.2)

Point-cloud 2D symbology can apply elevation shading per layer instead of as a
map-wide effect, preventing unrelated map elements from being blended.

### Point-cloud color expressions (4.2)

Renderers can modify base color with any point attribute and `@point_color`.
Arithmetic is channel-by-channel on RGBA. Multiplication accepts the color on
either side; other operators require it on the left:

```qgis
@point_color * (@intensity / 65535)
```
