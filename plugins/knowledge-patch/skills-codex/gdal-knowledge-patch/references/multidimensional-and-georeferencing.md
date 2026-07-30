# Multidimensional Data and Georeferencing

Multidimensional arrays, CRS handling, coordinate operations, geolocation, dates, and scientific data.

## Absent GeoHEIF geotransforms

*Batch 3.13.1*

A GeoHEIF dataset that has no geotransform no longer reports one.

## Closed polygons after polar reprojection

*Batch 3.12.4*

`OGRGeometryFactory::transformWithOptions()` closes polygons produced by its polar-reprojection path, including when used with GEOS 3.15.

## Dataset capabilities and metadata

*Batch 3.11.0*

Driver metadata now reflects update capabilities, and `GDAL_DCAP_CREATE_SUBDATASETS` identifies drivers supporting `APPEND_SUBDATASET=YES`; `GDALMDArray::AsClassicDataset()` accepts `BAND_IMAGERY_METADATA` for per-band imagery metadata. `GDAL_CACHEMAX` accepts memory units, new built-in tile matrix sets include `WorldMercatorWGS84Quad`, `PseudoTMS_GlobalMercator`, and `GoogleCRS84Quad`, and raster APIs now reject `GDT_Unknown` and `GDT_TypeCount`.

## Dataset extent, overview, and window APIs

*Batch 3.12.0*

New APIs include `GDALDataset::GetLayerIndex()`, `GetExtent()`, `GetExtentWGS84LongLat()`, and `AddOverviews()`, plus `GDALRasterBand::IterateWindows()` and `SplitRasterIO()`. `GDALGetGDALPath()` exposes GDAL's installation path, and `GDALRescaleGeoTransform()` rescales a geotransform.

## Driver connection and subdataset parsing

*Batch 3.12.3*

GeoRaster preserves double quotes in database connection strings. `GDALGetSubdatasetInfo()` now handles netCDF subdataset names whose endpoint includes a port number.

## ESRI fallback in `importFromEPSG()`

*Batch 3.10.1*

`OGRSpatialReference::importFromEPSG()` tries an ESRI lookup when a code looks like an ESRI code and emits a warning when that fallback succeeds.

## Fake GeoServer CRS values in WFS

*Batch 3.12.2*

The WFS driver now skips the synthetic GeoServer CRS identifier `EPSG:404000`.

## Fractional seconds at the minute boundary

*Batch 3.11.2*

`OGRParseDate()` parses a seconds value of `59.999999` as `59.999` rather than rounding it to `60.0`.

## GCP transformer option handling

*Batch 3.12.2*

`GDALTransformer()` ignores `MAX_GCP_ORDER` when `METHOD=GCP_TPS`; with `METHOD=GCP_POLYNOMIAL`, negative `MAX_GCP_ORDER` values are sanitized.

## Geolocation, geometry, schema, and celestial-body APIs

*Batch 3.12.0*

The geolocation transformer adds `GEOLOC_NORMALIZE_LONGITUDE_MINUS_180_PLUS_180` to force longitude normalization. OGR adds envelope-to-geometry creation and constrained Delaunay triangulation, vector datasets expose `GetSpatialRef()`, schema overrides accept `*` layer matching and `srcType`/`srcSubType` matching, and CRS APIs can report celestial-body names.

## Georeferencing validation

*Batch 3.12.3*

The netCDF driver uses a stored `GeoTransform` attribute only when it is consistent with the dimension variables. The RPFTOC driver now georeferences polar zones correctly.

## GML and AIXM geometry handling

*Batch 3.10.1*

The GML driver supports AIXM `ElevatedCurve` and honors `SWAP_COORDINATES=YES` even when a geometry has no spatial reference system.

## GML time instants

*Batch 3.11.5*

The GML and GMLAS drivers support `gml:TimeInstantType`.

## GTI SRS and interleave behavior

*Batch 3.13.0*

GTI adds `SRS_BEHAVIOR=OVERRIDE|REPROJECT` and exposes `INTERLEAVE=BAND|PIXEL`, honoring band interleave during on-the-fly warping.

## GTI support for richer STAC GeoParquet metadata

*Batch 3.10.1*

GTI can use STAC GeoParquet without `assets.image.href`; it recognizes `assets.XXX.proj:epsg`, `assets.XXX.proj:transform`, `proj:code`, `proj:wkt2`, and `proj:projjson`. It also reads `eo:bands` for any asset name, all `common_names`, central wavelength and full-width-half-maximum metadata, scale and offset from `raster:bands`, exposes the `SRS` open option, and attaches a sample tile's color table to a single-band GTI dataset.

## GTI warp controls and alpha output

*Batch 3.12.3*

The GTI driver adds a `WARPING_MEMORY_SIZE` open option. Its on-the-fly reprojection no longer creates a destination alpha band when one is unnecessary.

## HDF4 GCP generation at nodata coordinates

*Batch 3.11.5*

When generating ground control points, the HDF4 driver skips longitude and latitude values at nodata locations.

## HDF5 and netCDF multidimensional reads

*Batch 3.11.4*

HDF5 multidimensional arrays can be read with non-default strides, and geolocation references from `.aux.xml` resolve correctly. For netCDF, `LIST_ALL_ARRAYS=YES` also works when the dataset has no two-dimensional array.

## HDF5 swath geolocation metadata

*Batch 3.12.1*

For swath geolocation fields, HDF5 reports `GEOLOCATION` metadata instead of exposing those fields as ground control points.

## Homography overviews and viewshed value ranges

*Batch 3.12.3*

Homography GCP transformations now apply the correct scaling factor on overviews. Viewshed DEM and GROUND modes also accept values outside the `Byte` range.

## Invalid GCP-derived geotransforms

*Batch 3.10.2*

`GDALGCPsToGeoTransform()` now returns `FALSE` when it generates an invalid geotransform, allowing callers to reject the conversion instead of using an invalid result.

## JGD2024 in GML

*Batch 3.11.4*

The GML driver recognizes the JGD2024 coordinate reference system used by recent Japanese Fundamental Geospatial Data.

## Kerchunk Parquet reference stores

*Batch 3.12.1*

The Zarr driver restores an affected way of opening Kerchunk Parquet reference stores.

## Leap-second date parsing

*Batch 3.11.5*

`OGRParseDate()` accepts timestamps containing leap seconds.

## Missing Kerchunk targets

*Batch 3.11.5*

The Zarr driver now reports an error when a JSON/Kerchunk reference store points to a file that cannot be opened.

## Multiband MiraMon geotransforms

*Batch 3.12.2*

The MiraMonRaster driver now reports the correct dataset geotransform when a dataset has several bands.

## Multidimensional bridging and raw-block discovery

*Batch 3.12.0*

`GDALDataset::AsMDArray()` exposes a classic dataset as a multidimensional array, while `GDALMDArray::GetRawBlockInfo()` reports raw block information in HDF5, netCDF, Zarr, and VRT. Extended data types can expose raster attribute tables, groups can enumerate data types, and classic-dataset views can source band metadata from fully qualified attributes.

## Multidimensional reverse slicing

*Batch 3.11.5*

`CreateSlicedArray()` now slices a dimension's indexing variables as well as its data. One-element dimensions work with `GetView(["::-1"])`, and `VRTMDArraySourceFromArray::Read()` handles negative steps correctly.

## netCDF axis discovery

*Batch 3.11.2*

The netCDF driver recognizes the axis of `rhos` variables in PACE OCI products and can use a geolocation array to detect X and Y axes in three-dimensional variables.

## netCDF extra-dimension reporting

*Batch 3.10.1*

The netCDF driver adds the `GDAL_NETCDF_REPORT_EXTRA_DIM_VALUES` configuration option for reporting extra-dimension values.

## New raster data-source drivers

*Batch 3.13.0*

New read-only drivers expose E57 two-dimensional images and CPHD data through the multidimensional API. JP2GROK provides JPEG 2000 reading and writing through the AGPLv3-licensed Grok toolkit.

## PNG, WEBP, and Zarr access

*Batch 3.12.0*

PNG reads and writes background color through the `BACKGROUND_COLOR` dataset metadata item and accepts `ZLEVEL=0` for uncompressed output. WEBP supports `.wld` worldfiles, while Zarr can open `.zarray`, `.zgroup`, `.zmetadata`, and `zarr.json` files directly.

## Raster reads at edges, strides, and large sizes

*Batch 3.13.2*

Pansharpening can read a small window at a raster edge without a window error. Sliced multidimensional arrays compute correct parent bounds for `IAdviseRead()` with a step other than one, and block-based `RasterIO()` avoids integer overflow on huge rasters.

## Raster SDK additions

*Batch 3.11.0*

New SDK facilities include `gdal::CXXTypeTraits<T>`, `gdal::GDALDataTypeTraits<T>`, `gdal_minmax_element.hpp`, `gdal::VectorX`, `GDALRasterComputeMinMaxLocation()`/`GDALRasterBand::ComputeMinMaxLocation()`, `GDALDataset::GeolocationToPixelLine()`, `GDALRasterBand::InterpolateAtGeolocation()`, `GDALTranspose2D()`, `GDALGroup::GetMDArrayFullNamesRecursive()`, `GDALIsValueInRangeOf()`, and `GDALRasterBand::SetNoDataValueAsString()`.

## Raster types, covariance, and multidimensional overviews

*Batch 3.13.0*

`GDT_UInt8` is the canonical unsigned eight-bit data type and `GDT_Byte` aliases it. C, C++, and Python gain inter-band covariance-matrix APIs, while multidimensional arrays gain `GetOverviewCount()` and indexed `GetOverview()` access.

## Reprojection in Arrow streams from warped layers

*Batch 3.10.1*

`OGRWarpedLayer` no longer takes its Arrow stream directly from the source layer, because doing so skipped reprojection. Arrow-stream reads from a warped layer therefore retain the layer's reprojection behavior.

## Rotated-latitude/longitude netCDF georeferencing

*Batch 3.11.1*

The netCDF driver reads the spatial reference and geotransform from a Rotated Latitude Longitude grid mapping even when it has no ellipsoid definition.

## Sentinel-2 geolocation with missing granules

*Batch 3.12.2*

Geolocation-enabled Sentinel-2 subdatasets now tolerate expected missing granules.

## Transformations involving ESRI authority codes

*Batch 3.11.2*

Coordinate transformation works when one input CRS carries a code labeled as EPSG that is actually an ESRI code.

## Vector schema, defaults, and format controls

*Batch 3.11.0*

CSV, GML, and SQLite accept the `OGR_SCHEMA` open option, GeoJSON adds `FOREIGN_MEMBERS=AUTO/ALL/NONE/STAC`, and newly created GeoPackages default to version 1.4. DXF creation can set `$INSUNITS` and `$MEASUREMENT` and handles MultiPoint output and WIPEOUT input; Shapefile conversion writes DateTime as ISO 8601 text, GMLAS can resolve CityGML 2.0 without `schemaLocation`, and TopoJSON reads a top-level `crs`.

## Zarr and multidimensional discovery

*Batch 3.11.0*

The Zarr driver supports the current Zarr v3 specification with `zstd`, Kerchunk JSON and Parquet reference stores, and the Zarr v2 `shuffle`, `quantize`, `fixedscaleoffset`, and `imagecodecs_tiff` codecs/filters; it also reports compressor, filters, and array dimensions. Zarr and netCDF add `LIST_ALL_ARRAYS` defaulting to `NO`, while netCDF can identify a geolocation array without a `coordinates` attribute and uses `GeoTransform` to preserve precision.
