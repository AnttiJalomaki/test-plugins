# Multidimensional and scientific data

Use this reference for the task areas below. Batch labels identify when each behavior entered the covered compatibility history.

## Multidimensional arrays and discovery

### Multidimensional reverse slicing

*Batch: 3.11.5*

`CreateSlicedArray()` now slices a dimension's indexing variables as well as its data. One-element dimensions work with `GetView(["::-1"])`, and `VRTMDArraySourceFromArray::Read()` handles negative steps correctly.

### Multidimensional bridging and raw-block discovery

*Batch: 3.12.0*

`GDALDataset::AsMDArray()` exposes a classic dataset as a multidimensional array, while `GDALMDArray::GetRawBlockInfo()` reports raw block information in HDF5, netCDF, Zarr, and VRT. Extended data types can expose raster attribute tables, groups can enumerate data types, and classic-dataset views can source band metadata from fully qualified attributes.

### Multivalue multidimensional conversion

*Batch: 3.12.1*

`gdal mdim convert` now accepts multiple values for `--group`, `--subset`, and `--scale-axes`.

### Raster types, covariance, and multidimensional overviews

*Batch: 3.13.0*

`GDT_UInt8` is the canonical unsigned eight-bit data type and `GDT_Byte` aliases it. C, C++, and Python gain inter-band covariance-matrix APIs, while multidimensional arrays gain `GetOverviewCount()` and indexed `GetOverview()` access.

## Zarr and reference stores

### Zarr and multidimensional discovery

*Batch: 3.11.0*

The Zarr driver supports the current Zarr v3 specification with `zstd`, Kerchunk JSON and Parquet reference stores, and the Zarr v2 `shuffle`, `quantize`, `fixedscaleoffset`, and `imagecodecs_tiff` codecs/filters; it also reports compressor, filters, and array dimensions. Zarr and netCDF add `LIST_ALL_ARRAYS` defaulting to `NO`, while netCDF can identify a geolocation array without a `coordinates` attribute and uses `GeoTransform` to preserve precision.

### Missing Kerchunk targets

*Batch: 3.11.5*

The Zarr driver now reports an error when a JSON/Kerchunk reference store points to a file that cannot be opened.

### PNG, WEBP, and Zarr access

*Batch: 3.12.0*

PNG reads and writes background color through the `BACKGROUND_COLOR` dataset metadata item and accepts `ZLEVEL=0` for uncompressed output. WEBP supports `.wld` worldfiles, while Zarr can open `.zarray`, `.zgroup`, `.zmetadata`, and `zarr.json` files directly.

### Kerchunk Parquet reference stores

*Batch: 3.12.1*

The Zarr driver restores an affected way of opening Kerchunk Parquet reference stores.

### Zarr v3 sharding, metadata, and georeferencing

*Batch: 3.13.0*

Zarr v3 gains read, update, and creation of consolidated metadata; read/write support for `sharding_indexed`, `crc32c`, and variable-length UTF-8; and NumPy datetime/timedelta extension types. Multiscales map to GDAL overviews, `spatial` and `proj` conventions can be read or written with `GEOREFERENCING_CONVENTION=SPATIAL_PROJ`, and multidimensional overview building supports arrays with more than two dimensions.

## Tile indexes and STAC

### GTI support for richer STAC GeoParquet metadata

*Batch: 3.10.1*

GTI can use STAC GeoParquet without `assets.image.href`; it recognizes `assets.XXX.proj:epsg`, `assets.XXX.proj:transform`, `proj:code`, `proj:wkt2`, and `proj:projjson`. It also reads `eo:bands` for any asset name, all `common_names`, central wavelength and full-width-half-maximum metadata, scale and offset from `raster:bands`, exposes the `SRS` open option, and attaches a sample tile's color table to a single-band GTI dataset.

### STAC 1.1 and STACIT identification

*Batch: 3.10.2*

The STACIT driver supports STAC 1.1. Its `Identify()` method accepts an item when at least two of `proj:transform`, `proj:bbox`, and `proj:shape` are present.

### Timestamp-based GTI overview refresh

*Batch: 3.11.1*

`gdaladdo --partial-refresh-from-source-timestamp` works with GTI datasets as well as VRT datasets.

### GTI unreadable-source failures

*Batch: 3.11.5*

A GTI raster read now fails when one of its sources is unreadable instead of allowing the failed source read to pass unnoticed.

### GTI SQL sources and GeoTIFF metadata tags

*Batch: 3.12.0*

GTI accepts a SQL request instead of only a layer or table name for selecting tile features, and STAC GeoParquet `s3://` references are translated to `/vsis3/`. GeoTIFF reads and writes the `GDAL_METADATA` TIFF tag, including supported `json:*` metadata domains.

### South-up and newer STAC data in GTI

*Batch: 3.12.1*

GTI accepts south-up tiles and automatically warps them north-up. For STAC GeoParquet, it recognizes `stac_extensions` as a marker and supports a top-level `bands` object plus the EO 2.0 extension; URL rewriting is limited to STAC collection catalogs.

### STACIT pagination requests

*Batch: 3.12.1*

The STACIT driver no longer sends an initial pagination request with an empty `{}` body, improving compatibility with services that reject such a body.

### GTI warp controls and alpha output

*Batch: 3.12.3*

The GTI driver adds a `WARPING_MEMORY_SIZE` open option. Its on-the-fly reprojection no longer creates a destination alpha band when one is unnecessary.

### GTI relative paths and masked overview reads

*Batch: 3.12.4*

Relative filenames in GTI XML or `.gti.gpkg` indexes are resolved relative to the main file. Downsampled requests on a GTI dataset with a mask band and overviews no longer fail with a `panBandMap[0]` missing-band error.

### GTI SRS and interleave behavior

*Batch: 3.13.0*

GTI adds `SRS_BEHAVIOR=OVERRIDE|REPROJECT` and exposes `INTERLEAVE=BAND|PIXEL`, honoring band interleave during on-the-fly warping.

## Scientific and product formats

### netCDF extra-dimension reporting

*Batch: 3.10.1*

The netCDF driver adds the `GDAL_NETCDF_REPORT_EXTRA_DIM_VALUES` configuration option for reporting extra-dimension values.

### Rotated-latitude/longitude netCDF georeferencing

*Batch: 3.11.1*

The netCDF driver reads the spatial reference and geotransform from a Rotated Latitude Longitude grid mapping even when it has no ellipsoid definition.

### S-102 products without uncertainty

*Batch: 3.11.1*

The S102 driver opens products that have no uncertainty component and retrieves nodata correctly when only a depth component is present.

### netCDF axis discovery

*Batch: 3.11.2*

The netCDF driver recognizes the axis of `rhos` variables in PACE OCI products and can use a geolocation array to detect X and Y axes in three-dimensional variables.

### HDF5 and netCDF multidimensional reads

*Batch: 3.11.4*

HDF5 multidimensional arrays can be read with non-default strides, and geolocation references from `.aux.xml` resolve correctly. For netCDF, `LIST_ALL_ARRAYS=YES` also works when the dataset has no two-dimensional array.

### HDF4 GCP generation at nodata coordinates

*Batch: 3.11.5*

When generating ground control points, the HDF4 driver skips longitude and latitude values at nodata locations.

### Scientific and hydrographic raster formats

*Batch: 3.12.0*

`DTED_ASSUME_COMPLIANT` opts out of the driver's DTED value conversion below `-16000`. PDS4 supports `Int64` and `UInt64` rasters plus hexadecimal constant values; S102 reads Edition 3.0, S104 and S111 read Edition 2.0, and the S10x drivers decode custom coordinate reference systems.

### HDF5 swath geolocation metadata

*Batch: 3.12.1*

For swath geolocation fields, HDF5 reports `GEOLOCATION` metadata instead of exposing those fields as ground control points.

### Top-level ISIS3 metadata from GeoTIFF

*Batch: 3.12.2*

For the `json:ISIS3` metadata domain, `GetMetadataItem(<top-level-key>, json:ISIS3)` now returns the requested subset rather than the complete JSON object.

### ISIS3 PVL arrays and repeated keywords

*Batch: 3.12.2*

ISIS3 PVL-to-JSON and JSON-to-PVL conversion now handles arrays whose values carry units and metadata containing repeated keywords.

### DIMAP2 coverage metadata

*Batch: 3.13.2*

The DIMAP2 driver reports `CLOUD_COVERAGE` and `SNOW_COVERAGE` metadata items.
