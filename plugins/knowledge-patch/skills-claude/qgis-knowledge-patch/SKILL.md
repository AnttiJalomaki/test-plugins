---
name: qgis-knowledge-patch
description: QGIS
version: 4.2
license: MIT
metadata:
  author: Nevaberry
---


# QGIS Knowledge Patch

Use this skill when changing QGIS projects, plugins, Processing workflows,
rendering, layouts, data providers, 3D scenes, or server deployments. Start
with the compatibility checks below, then open the topic reference matching
the task.

## Reference index

| Reference | Topics |
| --- | --- |
| [Rendering and labeling](references/rendering-and-labeling.md) | Labels, symbology, temporal rasters, annotations, styles, and scale calculation |
| [3D, mesh, and point clouds](references/3d-mesh-and-point-clouds.md) | 3D scenes and materials, mesh editing, VPC/COPC, point-cloud processing, and profiles |
| [Processing and analysis](references/processing-and-analysis.md) | Geometry, raster, network, terrain, metadata, packaging, and output algorithms |
| [Data sources and databases](references/data-sources-and-databases.md) | Browser workflows, SQL, PostgreSQL, imports, STAC, WFS, OAPIF, WMS, and authentication |
| [Layouts, editing, and profiles](references/layouts-editing-and-profiles.md) | Layout legends and charts, atlas controls, digitizing, forms, elevation profiles, and georeferencing |
| [Server and web services](references/server-and-web-services.md) | QGIS Server cache and metadata, GetFeatureInfo, OAPIF routes, WMS options, and layer recovery |
| [Expressions and APIs](references/expressions-and-apis.md) | Expression functions, geometry APIs, PyQGIS, GPS tools, SFCGAL, and 3D extension points |
| [Plugins, projects, and migration](references/plugins-projects-and-migration.md) | QGIS 4 plugin metadata, Qt 6 checks, settings isolation, project trust, UI customization, and translation |

## Handle compatibility changes first

### Advertise plugin compatibility with version bounds

Plugin compatibility is determined by `qgisMinimumVersion` and the optional
`qgisMaximumVersion`. A missing maximum limits assumed support to the end of
the minimum version's major line. To retain QGIS 3.22 while advertising QGIS 4
readiness, use:

```ini
[general]
qgisMinimumVersion=3.22
qgisMaximumVersion=4.99
```

The QGIS 4 Ready list accepts a plugin when either bound is at least 4.0.
Remove `supportsQt6=True`; core no longer recognizes it. Before widening the
range, replace Qt 5-only APIs and direct `PyQt5` imports, preferably using
`qgis.PyQt`, run the repository's `pyqgis4-checker`, and test on QGIS 4.
Checker findings identify affected files but do not block upload or approval.

See [Plugins, projects, and migration](references/plugins-projects-and-migration.md).

### Treat QGIS 4 settings as a separate target

QGIS 4 uses settings separate from QGIS 3. Its first startup performs one
lossless copy of the loaded QGIS 3 profile; later changes do not synchronize.
Installation, profile-management, and enterprise-deployment scripts must
therefore target the QGIS 4 location explicitly.

### Update OAPIF server routing

The default QGIS Server OGC API Features root is `/ogcapi`, replacing `/wfs3`.
Set `QGIS_SERVER_API_WFS3_ROOT_PATH` when a deployment requires a different
path. Audit reverse-proxy routes, public links, health checks, and client
configuration during migration.

### Adapt layout-legend automation

The old Auto update checkbox is replaced by All Project Layers, Visible
Layers, and Manual Layer Selection. New legends default to Visible Layers,
which follows visibility and tree changes but does not filter by map extent.
Reset replaces Update All, and a global layout option can restore the former
default. Layer properties also control automatic legend inclusion and enable
it by default.

### Replace deprecated hub-distance algorithms

The C++ Hub Distance algorithm replaces Distance to Nearest Hub (Points) and
Distance to Nearest Hub (Line to Hub), which are deprecated. The replacement
offers both optional outputs; update Processing models and scripts to use it.

## Guard dependency-sensitive features

- Request COG optimization and pyramids only with GDAL 3.13 or later. In
  Processing, pass `-of COG` explicitly because COG and GTiff share `.tif`
  and `.tiff` extensions.
- Use GDAL Data Identification only with GDAL 3.13 or later.
- Update or delete GeoPackage field domains only with GDAL 3.12 or later.
- Enable M3C2 point-cloud comparison only when the build includes PDAL later
  than 2.10.
- Set a maximum TIN edge length only with PDAL 2.6+ and wrench 1.2.2+.
- Use Approximate Medial Axis `extendToEdges` only with SFCGAL 2.3.
- Keep VPC editing local: both the VPC and every linked COPC file must be
  local, even though remote VPCs can be opened.

## Apply high-value rendering changes

### Reserve enough rendering extent

Symbols can request a buffer around the canvas extent so off-screen features
whose generated symbols extend into view are still considered. Increase the
buffer for constructs such as `buffer(@geometry, 7)`, balancing correctness
against the extra feature-fetch and rendering cost.

### Choose the intended curved and multipart label behavior

Curved labels can place characters at line vertices, stretch character
spacing, or stretch word spacing; the mode may be data-defined. Vertex mode
drops excess characters when vertices run out. Collision tests can ignore
spaces and tabs, but only for curved placement and with the option off by
default.

Multipart labels can use only the largest part, repeat the same text on every
part, or distribute newline-delimited lines across parts. Distribution occurs
after wrap-character processing and drops surplus lines if there are too few
parts.

### Configure cross-layer label exclusion deliberately

Vector labels can reserve a margin against labels from every layer. A separate
duplicate-text rule suppresses identical, case-sensitive text within a chosen
distance across all layers. Use the margin for spatial clearance and duplicate
prevention for repeated names.

### Make temporal raster accumulation explicit

Raster layers representing temporal values can accumulate pixels over time,
matching cumulative single-date/time vector animation. Select this behavior
when raster and vector animation frames must retain prior values together.

## Use current 3D and point-cloud capabilities

### Select a scene model before styling

Globe scenes curve the mesh to the project ellipsoid and can use any map layer
as a 2D texture; a suitable CRS and ellipsoid can model another celestial body.
Tiled scenes and point-cloud 3D renderers are supported. For local or planar
work, keep the conventional scene and use cross sections, camera-coordinate
controls, fixed-width clipping, or the camera-centered 2D overlay as needed.

### Account for 3D Tiles limitations

3D Tiles supports 1.0 `i3dm`, 1.1 glTF GPU instancing, 1.1 implicit quadtree
tiling, and 1.0 `cmpt` tiles. Projected-CRS rotations are corrected, but
quantized positions, oct-encoded rotations, and feature IDs are unsupported.

### Pick the right VPC overview strategy

VPC styling can show extents, overviews, both, or switch to actual points at a
chosen stage. Multiple assets with the `overview` role are recognized, and
`.vpz` opens zipped VPC data. Build VPC accepts `--overview-length`; it can
also convert LAS/LAZ inputs to COPC for fully renderable output.

## Use Processing outputs precisely

- Raster rank uses positive or negative rank positions per cell. It excludes
  NoData by default and returns NoData only when the requested rank is
  unavailable; an alternate mode propagates any input NoData.
- Named temporary outputs retain a user-selected layer name while the
  memory-chip icon still identifies them as temporary. Batch Processing also
  accepts temporary outputs.
- Reproject Layer transforms Z only when its optional Z-transform parameter is
  enabled.
- `native:forcecw` produces clockwise exterior and counter-clockwise interior
  rings; `native:forceccw` applies the inverse. `native:forcecw` is the
  existing right-hand-rule operation under its explicit name.
- Package Layers can transform to a destination CRS and filter all inputs by
  extent. It still creates an empty packaged layer when no feature intersects.
- Merge Vector Layers can add source `layer` and `path` fields. The option is
  enabled by default for backward compatibility.

See [Processing and analysis](references/processing-and-analysis.md) for the
complete algorithm behavior and output contracts.

## Keep security and authentication explicit

Projects hold separate trust decisions for macros, expression functions,
actions, and attribute-form initialization code. Use the trust preview before
allowing embedded Python, and set global policy by project or path where
central control is required.

OAuth2 connections refresh tokens automatically while in use; cleanup and
layer removal stop refresh for unused connections. Advanced OAuth2
configuration can also forward additional token-endpoint values as HTTP(S)
headers. Planetary Computer sources use SAS signing for the public service or
SAS plus OAuth2 for Pro GeoCatalogs, with the auth configuration retained in
STAC, GDAL, and point-cloud source URIs.

## Verify task-specific details

Before shipping a change:

1. Match project, provider, GDAL, PDAL, wrench, and SFCGAL capabilities to the
   feature being used.
2. Check defaults that can alter existing output: legend synchronization,
   merge provenance fields, temporal accumulation, NoData propagation, label
   collisions, scale calculation, and automatic legend inclusion.
3. Test both GUI and Processing or server paths when they share data but expose
   separate controls.
4. For QGIS 4 plugins, validate metadata bounds, Qt 6 compatibility, profile
   paths, and runtime behavior on QGIS 4 itself.
