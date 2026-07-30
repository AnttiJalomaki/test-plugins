# Data sources and databases

## Database imports and SQL

### Map and filter imported fields

Since 3.44, database imports can rename fields, select exact destination
types, change source expressions, exclude fields, and create new fields.
Imports can filter by extent, expression, or current selection and can
uppercase or lowercase field names.

Dragging one layer to a Browser data source opens controls for destination
name, replacement, primary key, geometry column, CRS, and table comment.
Dragging multiple layers still imports immediately. The single-layer dialog
does not support Oracle.

### Save and reuse SQL

Since 3.44, the Browser **Execute SQL** dialog can insert, save, and remove
stored queries in either the project or user profile. Query history is
available directly in the Browser workflow. **Execute SQL** and **Update SQL**
can save and load `.sql` files.

Since 3.42, supported layers can also execute SQL from their context menu in
the project table of contents.

### Load SQL Server query layers

Since 3.44, SQL Server queries can be loaded as map layers from the Browser.
The SQL for an existing query layer can also be updated.

## PostgreSQL and PostGIS

### Store PostGIS raster styles and hide overviews

Since 3.42, the PostgreSQL raster provider can save raster styles in the
PostGIS database. A connection can hide raster overview tables listed by the
PostGIS `raster_overviews` view from the Browser.

### Limit a connection to one schema

Since 3.44, a PostgreSQL connection can be restricted to one schema. The
restriction applies in both the Browser and data-source selector.

### Administer projects and tables in the Browser

Since 3.44, the Browser can rename, delete, duplicate, or move PostGIS-stored
QGIS projects to another schema. It can also move PostgreSQL tables between
schemas and rename table fields.

Since 4.0, supporting providers let the Browser edit table comments and create
or delete spatial indexes. PostgreSQL layer properties show the current user's
table privileges, estimated row count, and spatial-index information.

### Import and version PostgreSQL projects

Since 4.0, the Browser can save the open project to a PostgreSQL schema or
batch-import projects from a folder. Name collisions receive suffixes such as
`_1`. Stored projects can carry comments displayed in Browser tooltips.
PostgreSQL-stored projects can enable automatic history and use QGIS dialogs
to save, load, edit, and restore earlier versions.

## Web feature sources

### Choose WFS feature and request modes

Since 3.44, WFS connection URIs and the connection UI accept `featureMode`.
`default` preserves server behavior, `SimpleFeatures` simplifies returned
features, and `ComplexFeatures` disables simplification. Each WFS connection
can also use POST in place of the default GET.

### Select OAPIF feature formats

Since 4.0, OGC API Features connections can choose server-advertised formats
instead of always using GeoJSON. Options include GML with or without a
described schema or bulk-download link. GML schema handling can use the simple
parser or GDAL GMLAS according to feature mode.
`lastFeatureFormatEncoding` supplies the default for new connections.

### Use SensorThings 2.0

Since 4.2, SensorThings layers support version 2.0 and the Sensing, Sampling,
and Relations extensions. The Browser and Data Source Manager dynamically
detect service version and available extensions.

## Catalogs and cloud assets

### Search STAC catalogs and footprints

Since 3.42, the Data Source Manager STAC client can search catalogs, apply
advanced result filters, show or hide result footprints on the map, and
highlight the selected item's footprint.

### Open broader cloud STAC assets

Since 4.2, STAC can open cloud-optimized assets from Azure and Google storage
and formats beyond GeoTIFF, including JPEG 2000, TileDB, and point clouds. The
asset must have a `cloud-optimized` MIME label or a supported asset-type
declaration.

### Authenticate Planetary Computer sources

Since 4.0, the authentication manager supports SAS signing for the public
Microsoft Planetary Computer and SAS plus OAuth2 for Pro GeoCatalogs. The
configuration works with STAC, GDAL, and point-cloud layers and remains in
their data-source URIs.

## WMS and ESRI REST sources

### Propagate WMS-T dimensions

Since 3.44, WMS-T layer-tree groups can recursively compute and expose time
dimensions from child layers. Disabling the setting on a group also stops its
children's dimensions propagating to the parent. Group dimensions support OGC
WMS/ISO 8601 date ranges. A raster layer can use one fixed date/time to infer
both ends of its temporal range.

### Persist the preferred WMS image format

Since 4.0, WMS connections detect formats advertised by the server and
persist the preferred/default format in settings for later use.

### Use the changed ESRI REST Browser layout

Since 4.2, the Browser collapses duplicate FeatureServer-vector and
MapServer-raster entries into the FeatureServer item. Raster loading moves to
its context menu. The MapServer **All layers** pseudo-layer is likewise
replaced by a context-menu action on the map service.

## GeoPackage and geocoding

### Maintain GeoPackage field domains

Since 4.0, GeoPackage field domains can be updated or deleted when QGIS uses
GDAL 3.12 or later.

### Filter Nominatim results by country

Since 3.44, the Nominatim Geocoder Locator accepts a comma-separated list of
two-letter country codes and restricts results to those countries.

## OAuth2 request behavior

### Forward extra tokens as headers

Since 3.44, advanced OAuth2 configuration can attach additional tokens
returned by the token endpoint as HTTP(S) request headers for any OAuth2
service.

### Refresh active tokens automatically

Since 4.0, OAuth2 connections automatically refresh tokens while in use.
Periodic cleanup and layer removal stop refresh activity for unused
connections.
