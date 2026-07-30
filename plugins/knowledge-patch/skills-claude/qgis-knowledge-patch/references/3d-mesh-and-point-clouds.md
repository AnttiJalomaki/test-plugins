# 3D, mesh, and point clouds

## Mesh rendering, editing, and export

### Edit mesh topology and elevation

Since 3.42, adding a mesh vertex can Delaunay-refine adjacent triangles by
flipping nonconforming edges. Selection actions cover all vertices or only
isolated vertices. New-vertex Z policies can prefer the mesh and then a Z
widget or terrain, always use project terrain, or always use the Z widget.
Selected vertices can also infer Z from project terrain.

### Manage mesh dataset groups

Since 3.42, externally added dataset groups may share names; QGIS numbers them
automatically to disambiguate the names. These added groups can be removed,
but groups belonging to the original mesh source cannot.

### Export a mesh surface

Since 3.42, **Mesh: Surface to Polygon** exports the mesh surface as a
MultiPolygon layer.

## Scene construction and navigation

### Cut 3D cross sections

Since 3.44, the cross-section tool captures start, end, and thickness points
from the 2D canvas, filters the 3D scene to that possibly rotated region, and
moves the camera to a side view. Toggling the section does not reload the
whole scene.

Since 4.0, sections can use a fixed editable width and can be nudged left or
right. Scene export can omit terrain, extruded polygons can include floors,
and a 3D view can display a camera-centered 2D map overlay with an optional
camera frustum.

### Use globe scenes

Since 3.44, globe mode bends the scene mesh to the project ellipsoid. Any map
layer can provide the 2D texture, and tiled-scene and point-cloud 3D renderers
are supported. A suitable project CRS and ellipsoid can represent another
celestial body instead of Earth.

### Set camera coordinates and controls

Since 4.2, the camera dialog can set target XYZ in map-CRS coordinates
together with pitch, heading, and distance. Optional live update pushes edits
to the view, while displayed values always follow camera movement. Vertical
axis inversion is independently configurable for walk dragging,
captured-mouse walk mode, and terrain mode.

### Export STL scenes

Since 4.2, 3D scenes export to STL as well as OBJ. STL is simpler and does not
preserve textures.

## 3D vector rendering and materials

### Use categorized and rule-based 3D renderers

Since 4.2, vector 3D symbology supports categorized and rule-based renderers
with controls modeled on the corresponding 2D renderers.

### Configure physically based materials

Since 4.2, the physically based material accepts base-color, metalness,
roughness, and ambient-occlusion texture maps. Metal-rough materials also
support opacity, a solid emission color and strength, and data-defined base
and emission colors. Both metal-rough and Phong textures expose data-defined
scale, rotation, and offset. Save 3D materials as tagged or favorited presets
in the style database.

### Set model axes

Since 4.2, 3D point-model symbols can explicitly choose up and forward axes
instead of assuming Z-up and Y-forward. This avoids corrective manual
rotations that interfere with reusable rotation, scale, and data-defined
settings.

### Add environmental lighting and effects

Since 4.2, a cube-map skybox can dynamically generate environmental lighting
for physically based materials. This is optional and does not apply to
fixed-gradient backgrounds. 3D Effects also provides tone mapping, exposure,
gamma, light bloom, global MSAA, and configurable gradient backgrounds.

## Tiled scenes and 3D Tiles

### Open Esri I3S

Since 4.0, the tiled-scene provider opens I3S 1.7+ `3DObject` and
`IntegratedMesh` content from ArcGIS REST services or local SLPK files in 2D
and 3D. It supports global EPSG:4326 data and local data in projected CRSs.

### Render expanded 3D Tiles

Since 4.2, QGIS renders instanced meshes from 3D Tiles 1.0 `i3dm` and 1.1 glTF
GPU instancing in both 2D and 3D, including rotation correction for projected
CRSs. Quantized positions, oct-encoded rotations, and feature IDs are not
supported. It also reads 1.1 implicit tiling with quadtree subdivision and 1.0
composite `cmpt` tiles.

## Virtual point clouds and COPC

### Select VPC overview behavior

Since 3.42, a VPC renders its overview while zoomed out when one exists, or
renders extents when none exists. Styling can force extents only, overview
only, or both.

Since 4.0, styling can also choose how early actual points replace extents or
overviews. Build VPC can convert LAS/LAZ inputs to COPC so the result is fully
renderable. Remote VPCs can be opened directly, but editing requires the VPC
and all linked COPC files to be local.

### Use multiple overviews and zipped VPC

Since 4.2, Build VPC accepts an optional `--overview-length`. Readers recognize
every asset with the `overview` role, regardless of asset ID, and render
multiple overviews when zoomed out. `QgsPointCloudLayer.overviews()` and
`QgsVirtualPointCloudProvider.overviews()` return overview lists. Open zipped
VPC datasets from `.vpz` files.

### Produce COPC from Processing

Since 3.44, PDAL Processing algorithms can write Cloud Optimized Point Cloud
outputs directly.

## Point-cloud editing, styling, and analysis

### Edit point attributes in 3D

Since 3.44, select a point-cloud attribute and target value in a 3D view, then
select points with polygon, paintbrush, or above/below-line tools. An
expression filter can restrict which selected points are modified.

### Compare point clouds with M3C2

Since 4.0, **Compare Point Clouds** computes signed multiscale distances along
locally estimated surface normals through PDAL `filters.m3c2`. It is available
only when the QGIS build includes PDAL later than 2.10.

### Normalize and clean point clouds

Since 4.0, **Height Above Ground** adds a `HeightAboveGround` attribute or
replaces Z, using nearby or triangulated ground points classified as class 2.
Processing also includes SMRF ground classification, statistical and radius
noise filters, and point-cloud translation, rotation, and scaling.

### Limit TIN edges

Since 4.0, PDAL **Export to Raster (TIN)** can omit triangles whose edges
exceed a maximum length. This parameter requires PDAL 2.6+ and wrench 1.2.2+.

### Apply elevation shading per layer

Since 4.2, point-cloud 2D symbology can apply elevation shading per layer
instead of using the map-wide effect, preventing unrelated map content from
being blended into the shading.

### Modify point colors with expressions

Since 4.2, point-cloud renderers can modify their base color with expressions
using any point attribute and `@point_color`. Arithmetic operates
channel-by-channel on RGBA. Multiplication accepts the color on either side;
other operators require it on the left:

```qgis
@point_color * (@intensity / 65535)
```

## Profiles and 3D extension

### Render continuous point-cloud profile lines

Since 4.0, elevation profiles can render point clouds as continuous elevation
lines rather than points; tolerance controls sparse results. A lockable
distance-to-elevation scale ratio can replace the normal 1:1 navigation ratio.

### Show elevation profiles in 3D

Since 4.2, **Show Profile in 3D Views** displays an elevation curve in 3D,
derives Z limits from data within the curve, and links cursor position with
line or polygon rubber bands.

### Extend PyQGIS 3D tools

Since 4.0, plugins can derive custom canvas tools from `Qgs3DMapTool`, apply
the cross-section tool's four clipping planes, and call
`Qgs3DMapCanvas.castRay()` to obtain and manage 3D hits using `QgsRay3D`.
