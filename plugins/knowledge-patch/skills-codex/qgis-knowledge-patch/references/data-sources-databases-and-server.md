# Data Sources, Databases, and QGIS Server

## Catalogs, services, and connection behavior

### Snapping in the Georeferencer (3.42)

The Georeferencer includes snapping options and the Advanced Digitizing panel
for placing reference points against existing geometry.

### STAC catalog search and footprints (3.42)

The Data Source Manager STAC client can search catalogs, apply advanced result
filters, show or hide result footprints on the map, and highlight the selected
item's footprint.

### Nominatim country filtering (3.44)

The Nominatim Geocoder Locator can restrict results to countries using a
comma-separated list of two-letter country codes.

### WFS feature and request modes (3.44)

WFS connection URIs and UI accept `featureMode`: `default` retains server
behavior, `SimpleFeatures` simplifies returned features, and
`ComplexFeatures` disables simplification. A connection can use POST instead
of the default GET.

### Selectable OAPIF feature formats (4.0)

OGC API Features connections can select advertised formats rather than always
using GeoJSON. Choices include GML with or without a described schema or
bulk-download link. GML schema handling can use the simple parser or GDAL
GMLAS according to feature mode. `lastFeatureFormatEncoding` supplies the
default for new connections.

### Planetary Computer authentication (4.0)

The authentication manager supports SAS signing for the open Microsoft
Planetary Computer and SAS plus OAuth2 for Pro GeoCatalogs. The auth
configuration works with STAC, GDAL, and point-cloud layers and is carried in
their data-source URIs.

### Persistent WMS image-format selection (4.0)

WMS connections detect server-advertised image formats and persist the
preferred/default format for later use.

### SensorThings 2.0 (4.2)

SensorThings layers support version 2.0 and the Sensing, Sampling, and
Relations extensions. The Browser and Data Source Manager detect service
version and extensions dynamically.

### Changed ESRI REST Browser layout (4.2)

The Browser collapses duplicate FeatureServer-vector and MapServer-raster
entries into the FeatureServer item; raster loading moves to its context menu.
The MapServer `All layers` pseudo-layer is replaced by a context-menu action on
the map service.

### Broader cloud STAC assets (4.2)

STAC can open cloud-optimized assets from Azure and Google storage and formats
beyond GeoTIFF, including JPEG 2000, TileDB, and point clouds, when the asset
has a `cloud-optimized` MIME label or a supported asset-type declaration.

## Temporal services and authentication

### Temporal controls for WMS-T groups and rasters (3.44)

WMS-T layer-tree groups recursively compute and expose a time dimension from
children. Disabling the feature on a group prevents its children's dimensions
from propagating to the parent. Group dimensions can contain OGC WMS/ISO 8601
date ranges; a raster can use one fixed datetime to infer both temporal ends.

### Extra OAuth2 tokens as headers (3.44)

Advanced OAuth2 configuration can attach additional values returned by the
token endpoint as HTTP(S) request headers for any OAuth2 service.

### OAuth2 token auto-refresh (4.0)

OAuth2 connections refresh tokens automatically while in use. Periodic
cleanup and layer removal stop refresh for unused connections.

## Database import and Browser administration

### PostGIS raster storage controls (3.42)

The PostgreSQL raster provider can save raster styles in PostGIS. A connection
can hide raster overview tables listed by the `raster_overviews` view from the
Browser.

### Database import mapping and filtering (3.44)

Imports can rename fields, select exact destination types, change source
expressions, exclude or add fields, filter by extent/expression/selection, and
uppercase or lowercase field names. Dragging one layer to a Browser data
source opens controls for destination name, replacement, primary key, geometry
column, CRS, and table comment. Multi-layer drag imports immediately. This
dialog does not support Oracle.

### SQL query persistence (3.44)

The Browser Execute SQL dialog can insert, save, and remove queries stored in
the project or user profile, and exposes query history in the Browser
workflow. Execute SQL and Update SQL can save and load `.sql` files.

### PostgreSQL Browser management (3.44)

The Browser can rename, delete, duplicate, or move PostGIS-stored QGIS
projects to another schema. It can move PostgreSQL tables between schemas and
rename their fields.

### Single-schema PostgreSQL connections (3.44)

A PostgreSQL connection can be restricted to one schema, limiting the Browser
and data-source selector to matching tables.

### SQL Server query layers (3.44)

SQL Server queries can be loaded as map layers from the Browser, and the SQL
of existing query layers can be updated.

### Browser database administration (4.0)

Supporting providers let the Browser edit table comments and create or delete
spatial indexes. PostgreSQL layer properties show current-user table
privileges, estimated row count, and spatial-index information.

### GeoPackage field-domain maintenance (4.0)

GeoPackage field domains can be updated or deleted with GDAL 3.12 or later.

### PostgreSQL project import and versioning (4.0)

The Browser can save the open project to a PostgreSQL schema or batch-import
projects from a directory, adding suffixes such as `_1` on collisions. Stored
projects can have comments shown in Browser tooltips. PostgreSQL projects can
enable automatic history and use QGIS dialogs to save, load, edit, and restore
earlier versions.

## QGIS Server configuration and output

### Configurable server project cache (3.44)

`QGIS_SERVER_PROJECT_CACHE_SIZE` configures the QCache cost used for the
server project cache instead of the former hardcoded value.

### Server metadata on layer-tree groups (3.44)

Groups can publish keywords, data URL and format, attribution title and URL,
metadata URLs, and legend URL and format through GetCapabilities, in addition
to short name, title, and abstract. If no legend URL is set, a legend is
generated by default.

### HTML GetFeatureInfo maptip-only mode (4.0)

A project setting can make HTML GetFeatureInfo use only the layer maptip. Its
WMS vendor parameter is `WITH_MAPTIP=HTML_FI_ONLY_MAPTIP`.

### Retrying invalid QGIS Server layers (4.0)

`QGIS_SERVER_RETRY_BAD_LAYERS=true` retests layers previously accepted as bad
on every request and restores them when their dependency recovers.

### Changed OAPIF server root (4.0)

The default OAPIF root changes from `/wfs3` to `/ogcapi`. Set
`QGIS_SERVER_API_WFS3_ROOT_PATH` when another path is required.

### FlatGeobuf OAPIF export (4.2)

QGIS Server's OGC API Features service can export FlatGeobuf.
