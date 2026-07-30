# Raster processing and formats

Use this reference for the task areas below. Batch labels identify when each behavior entered the covered compatibility history.

## Warping, reprojection, and georeferencing

### Invalid GCP-derived geotransforms

*Batch: 3.10.2*

`GDALGCPsToGeoTransform()` now returns `FALSE` when it generates an invalid geotransform, allowing callers to reject the conversion instead of using an invalid result.

### Warping empty source windows

*Batch: 3.10.3*

Warping with the MEM driver now handles an empty source window correctly when the nodata value is nonzero.

### Warp and coordinate-operation controls

*Batch: 3.11.0*

The transformer adds a Homography type, while the warper adds `MODE_TIES`, uses source-pixel coverage for mode resampling, and permits a mode of `-1`. Transformer options now include `ALLOW_BALLPARK=NO`, `ONLY_BEST=AUTO/YES/NO`, source/destination axis-mapping controls, and `HEIGHT_DEFAULT` as the fallback RPC height; `ogr2ogr -ct_opt` exposes the ballpark, best-operation, and differing-operation-warning controls.

### Reprojecting directly to COG

*Batch: 3.11.2*

`gdalwarp` can again reproject into Cloud Optimized GeoTIFF output, fixing a regression introduced in 3.11.0.

### Transformations involving ESRI authority codes

*Batch: 3.11.2*

Coordinate transformation works when one input CRS carries a code labeled as EPSG that is actually an ESRI code.

### Large, global, and TPS warps

*Batch: 3.11.4*

Warping large rasters no longer fails in cases such as globally extensive WMTS inputs, and a whole longitude range of at least 360 degrees is no longer given an inappropriate `CENTER_LONG` when targeting Web Mercator. TPS warping now defaults to `-wo SOURCE_EXTRA=5`.

### Polar-to-geographic reprojection

*Batch: 3.11.5*

Geometry reprojection from a polar CRS to geographic coordinates is corrected in both the core transformation path and `gdal vector reproject`.

### Reprojection, resizing, tiling, and viewshed controls

*Batch: 3.12.0*

`gdal raster reproject` adds `-j`/`--num-threads` and defaults to `ALL_CPUS`, while `gdal raster resize` adds `--resolution`. Raster tiling supports `--parallel-method=fork` on non-Windows systems or `spawn`, emits `stacta.json`, and can terminate a pipeline; viewshed adds angular, pitch, and minimum-distance masking.

### Sum-resampled warps

*Batch: 3.12.1*

`gdalwarp -r sum` no longer introduces artifacts related to chunked processing.

### GCP transformer option handling

*Batch: 3.12.2*

`GDALTransformer()` ignores `MAX_GCP_ORDER` when `METHOD=GCP_TPS`; with `METHOD=GCP_POLYNOMIAL`, negative `MAX_GCP_ORDER` values are sanitized.

### Multiband MiraMon geotransforms

*Batch: 3.12.2*

The MiraMonRaster driver now reports the correct dataset geotransform when a dataset has several bands.

### Warp data types

*Batch: 3.12.3*

`GDALWarpResolveWorkingDataType()` now examines band data types before falling back to `UInt8`, and nearest-neighbor warping has a dedicated `Int8` path. Signed-byte inputs therefore no longer depend on byte-oriented working-type behavior.

### Homography overviews and viewshed value ranges

*Batch: 3.12.3*

Homography GCP transformations now apply the correct scaling factor on overviews. Viewshed DEM and GROUND modes also accept values outside the `Byte` range.

### Reprojected and non-square-pixel extents

*Batch: 3.12.3*

`gdaltindex` now uses GDALWarp when computing reprojected extents. `gdal2tiles` also computes the correct extent for source rasters with non-square pixels.

### Georeferencing validation

*Batch: 3.12.3*

The netCDF driver uses a stored `GeoTransform` attribute only when it is consistent with the dimension variables. The RPFTOC driver now georeferences polar zones correctly.

### Multithreaded warp interruption

*Batch: 3.12.4*

Multithreaded warps detect progress interruption more reliably, and warping initiated from a worker thread avoids a potential deadlock.

### Resampling with NaN nodata

*Batch: 3.12.4*

Bilinear, cubic, cubic-spline, and Lanczos resampling now handle NaN values correctly when the band's nodata value is also NaN.

### Warp destination initialization

*Batch: 3.13.0*

`gdalwarp` now fails when `INIT_DEST=NO_DATA` is requested without a nodata value. The new `RESET_DEST_PIXELS=YES|NO` warp option can completely reset an existing destination to destination nodata or zero.

### Absent GeoHEIF geotransforms

*Batch: 3.13.1*

A GeoHEIF dataset that has no geotransform no longer reports one.

### Vertical-shift unit metadata

*Batch: 3.13.1*

For a 3D-to-3D vertical-shift warp, `gdalwarp` no longer copies the source unit type to the output file.

## VRT, pansharpening, and derived bands

### VRT processed-dataset scaling

*Batch: 3.10.1*

Processed VRT datasets now read scale and offset from their source dataset.

### Pansharpening nearly aligned inputs

*Batch: 3.10.3*

Pansharpening no longer reports I/O errors when the extents of the panchromatic and multispectral bands differ by less than one multispectral-band resolution.

### Embedded resources and VRT expression dependencies

*Batch: 3.11.0*

CMake adds `EMBED_RESOURCE_FILES` and `USE_ONLY_EMBEDDED_RESOURCE_FILES` for compiling resource files into libgdal. `muparser` is strongly recommended as a build and runtime dependency for C++ VRT expressions; header-only `exprtk` may be added alongside it for advanced expressions, at an approximately 8 MB library-size cost.

### Richer VRT composition

*Batch: 3.11.0*

VRT pixel functions can evaluate arbitrary expressions, reclassify values, and apply `mul` or `sum` with a constant factor to one band. A `<SimpleSource>` or `<ComplexSource>` may embed a `<VRTDataset>` instead of naming a source file, and processed VRTs gain an `OutputBands` element for declaring output count and data types.

### Complete VRT overview exposure

*Batch: 3.11.2*

A single-source VRT exposes all source overviews regardless of their size. `VRTPansharpen` also tolerates source bands with differing numbers of overviews when generating virtual overviews.

### Nodata on pansharpened VRT overviews

*Batch: 3.11.5*

`VRTPansharpenedRasterBand` overview bands now inherit the nodata value of the full-resolution band.

### VRT derived-band functions and expressions

*Batch: 3.12.0*

VRT pixel functions add `mean`, `median`, `geometric_mean`, `harmonic_mean`, `mode`, `argmin`, and `argmax`, and now account for nodata; `min` and `max` accept an optional `k` constant. Muparser expressions add `fmod`, derived-band expressions expose `_CENTER_X_` and `_CENTER_Y_`, and `vrt://` accepts a `transpose` option.

### `ComplexSource` calculation types

*Batch: 3.12.1*

`gdal raster calc` and `VRTDerivedRasterBand` now use the correct computation and transfer data types with a `ComplexSource`.

### Pansharpened VRT serialization and orientation

*Batch: 3.12.2*

Pansharpened VRTs now serialize correctly when the panchromatic and multispectral bands have different extents. The vertical-orientation test for input datasets is also corrected.

### Strided VRT derived-band reads

*Batch: 3.12.3*

`VRTDerivedRasterBand::IRasterIO()` correctly zero-initializes output buffers when line spacing differs from pixel spacing multiplied by the buffer width.

### Implicit VRT derived-band overviews

*Batch: 3.12.4*

`VRTDerivedRasterBand` now creates implicit overviews correctly.

### VRT derived functions and block selection

*Batch: 3.13.0*

VRT derived bands add `area`, `quantile`, and `round` pixel functions. The `vrt://` connection protocol also accepts a `block` option.

### Out-of-range nodata warnings in separate VRTs

*Batch: 3.13.1*

`gdalbuildvrt -separate` warns when a nodata value is outside the range of the target band type.

## Raster analysis, masks, and overviews

### Raster utility options and stricter behavior

*Batch: 3.11.0*

`gdalbuildvrt` adds `-co` and `-resolution same|compatible`; `gdaldem` derives scale from the CRS and adds `-xscale`/`-yscale`; `gdallocationinfo` can query corners; `rgb2pct` adds `--creation-option`; `gdal2xyz` can write to VSI paths; and `gdalenhance` is now installed and documented. `gdal_translate -projwin` includes partially covered pixels and transforms the full bounds, translation and warping reject invalid numeric options, nodata is copied only when exactly representable, polygonized contours omit min/max fields, and `gdal2tiles` applies source nodata even without reprojection.

### Raster reads with masks, NaNs, and constant histograms

*Batch: 3.11.4*

`GDALNoDataMaskBand::IRasterIO()` no longer corrupts Byte-band reads when `nLineSpace > nBufXSize`. Overview mode resampling accounts for `NaN` in `Float16` and `CFloat16`, while `GetDefaultHistogram()` handles constant-valued non-Byte data where `min == max`.

### COG translation with a selected mask band

*Batch: 3.11.5*

`gdal_translate -of COG -b 1 -b 2 -b 3 -b mask ...` can translate an RGB dataset with overviews without crashing; the selected mask becomes a regular output band tagged as alpha.

### Zonal statistics at and beyond raster bounds

*Batch: 3.12.1*

`GDALZonalStats` handles affected polygons outside the raster extent, while `gdal raster zonal-stats` avoids integer overflow for geometries with huge coordinate values.

### All-nodata contour inputs

*Batch: 3.12.2*

Contouring an all-nodata raster now succeeds with an empty output layer instead of emitting an error.

### RMS overview normalization

*Batch: 3.12.3*

RMS overview resampling uses a corrected normalization formula, changing values produced by affected overviews.

### Palette, zonal, and rasterization controls

*Batch: 3.13.0*

`gdal raster rgb-to-palette` adds `--output-nodata`, `--no-dither`, and `--bit-depth`; zonal statistics accepts `--include-field ALL|NONE`, `--include-geom`, and an output layer. Rasterization can derive one output size from the other size and the input extent when one size is zero.

### Current GMT color tables

*Batch: 3.13.1*

`gdal raster color-map` accepts current GMT `.cpt` color-table files.

### Automatic source-overview selection for tiling

*Batch: 3.13.1*

`gdal raster tile` now selects an appropriate source overview automatically.

### BigTIFF nodata values in LIBERTIFF

*Batch: 3.13.1*

LIBERTIFF correctly reads a BigTIFF nodata value whose string representation occupies four through eight bytes.

### Masked naked Lerc2 files

*Batch: 3.13.1*

The MRF driver can decode naked Lerc2 files containing masks when built with liblerc 3.0 or newer.

### Raster reads at edges, strides, and large sizes

*Batch: 3.13.2*

Pansharpening can read a small window at a raster edge without a window error. Sliced multidimensional arrays compute correct parent bounds for `IAdviseRead()` with a step other than one, and block-based `RasterIO()` avoids integer overflow on huge rasters.

### Exact integer nodata statistics

*Batch: 3.13.2*

`ComputeRasterMinMax()` and `GetHistogram()` now require an exact integer match when excluding a nodata value, changing results in cases that previously treated a different integer as nodata.

## Raster formats and driver behavior

### JPEG XL from DNG in GeoTIFF

*Batch: 3.10.1*

The GTiff driver supports `Float16` and TIFF compression value `52546`, the JPEG XL encoding defined by DNG 1.7.

### ESRI fallback in `importFromEPSG()`

*Batch: 3.10.1*

`OGRSpatialReference::importFromEPSG()` tries an ESRI lookup when a code looks like an ESRI code and emits a warning when that fallback succeeds.

### AVIF images larger than 10 MB

*Batch: 3.10.3*

The AVIF driver can now read images larger than 10 MB.

### GRIB2 Transverse Mercator variants

*Batch: 3.10.3*

The GRIB2 driver now reads Transverse Mercator definitions with negative easting/falsing values or a scale factor other than `0.9996`.

### Immediate reads of compressed multithreaded GeoTIFF output

*Batch: 3.10.3*

A compressed GeoTIFF created in multithreaded mode can now be read immediately after creation, fixing a regression introduced in 3.10.1.

### WEBP-compressed MBTiles updates

*Batch: 3.10.3*

The MBTiles driver can now update datasets that use WEBP compression.

### New data-source drivers

*Batch: 3.11.0*

The read-only OGR ADBC driver can access DuckDB or Parquet datasets when libduckdb is installed, while LIBERTIFF provides a native thread-safe read-only GeoTIFF reader. Read-only RCM and AIVector drivers are also new.

### Removed drivers and writers

*Batch: 3.11.0*

Removed raster drivers are BLX, BT, CTable2, ELAS, FIT, GSAG, GSBG, JP2Lura, OZI OZF2/OZFX3, Rasterlite v1, R object `.rda`, RDB, SDTS, SGI, XPM, and DIPex; removed vector drivers are Geoconcept Export, OGDI, SDTS, SVG, Tiger, and UK .NTF. Write support was removed from Interlis 1/2, ADRG, PAux, MFF, MFF2/HKV, LAN, NTv2, BYN, USGSDEM, and ISIS2.

### Other upgrade compatibility changes

*Batch: 3.11.0*

The OpenCL warper and the unofficial `gdalwarpsimple` and `ogrdissolve` applications were removed; the OGR `Memory` driver is deprecated and aliases the unified `MEM` driver, and the shared-library major version was bumped. FileGDB update and creation now route through OpenFileGDB, while PDF creation no longer supports `GEO_ENCODING=OGC_BP`.

### Dataset capabilities and metadata

*Batch: 3.11.0*

Driver metadata now reflects update capabilities, and `GDAL_DCAP_CREATE_SUBDATASETS` identifies drivers supporting `APPEND_SUBDATASET=YES`; `GDALMDArray::AsClassicDataset()` accepts `BAND_IMAGERY_METADATA` for per-band imagery metadata. `GDAL_CACHEMAX` accepts memory units, new built-in tile matrix sets include `WorldMercatorWGS84Quad`, `PseudoTMS_GlobalMercator`, and `GoogleCRS84Quad`, and raster APIs now reject `GDT_Unknown` and `GDT_TypeCount`.

### COG and TIFF creation behavior

*Batch: 3.11.0*

COG creation supports `INTERLEAVE=BAND` and `TILE`, notably for hyperspectral data. GTiff reads ArcGIS-style `.tif.vat.dbf` raster attribute tables, and GTiff, COG, and warping preserve premultiplied-alpha information from source TIFFs.

### Additional imagery-format capabilities

*Batch: 3.11.0*

HEIF gains tile reading, `CreateCopy()`, and read-only GeoHEIF support with libheif 1.19; AVIF gains read-only GeoHEIF support with the libavif development version current at release time, and JPEGXL reads Float16 as Float32. DIMAP exposes PNEO FWHM and RPC `HEIGHT_DEFAULT`, NITF represents SAR I/Q pairs as one complex band, Sentinel-2 recognizes `S2C_` names, and Leveller accepts document versions through 12.

### Restored GSBG raster support

*Batch: 3.11.1*

The GSBG driver for Golden Software Surfer Binary Grid 6.0 is restored after its removal in 3.11.0.

### FLIR thermal JPEG handling

*Batch: 3.11.1*

The JPEG driver reads FLIR thermal images stored as little-endian 16-bit PNG data. It exposes `IRWindowTransmission` separately instead of overwriting `IRWindowTemperature`, and corrects the metadata subdomain for `RelativeHumidity`.

### Restored GSAG raster support

*Batch: 3.11.2*

The GSAG driver for Golden Software ASCII Grid is available again after its removal in 3.11.0.

### WEBP-compressed RGBA in LIBERTIFF

*Batch: 3.11.2*

LIBERTIFF reads WEBP-compressed RGBA images even when a fully opaque tile or strip omits its alpha component.

### PNG caching without a band-one read

*Batch: 3.11.2*

The PNG driver correctly caches other bands even when reading does not begin with band 1.

### Fractional seconds at the minute boundary

*Batch: 3.11.2*

`OGRParseDate()` parses a seconds value of `59.999999` as `59.999` rather than rounding it to `60.0`.

### Restored BT raster support

*Batch: 3.11.4*

The BT driver is available again after its removal in 3.11.0.

### Complex COG and RGB-NIR GeoTIFF creation

*Batch: 3.11.4*

The COG driver can create datasets with complex data types. The GTiff driver can create R, G, B, NIR files without an explicit `PHOTOMETRIC` creation option.

### JPEG XL conversion and lossy-option diagnostics

*Batch: 3.11.4*

`gdal_translate non_byte.jxl byte.jxl -ot Byte` now converts JPEG XL data correctly. GTiff and COG emit warnings when `JXL_DISTANCE` or `JXL_ALPHA_DISTANCE` is used without `JXL_LOSSLESS=NO`.

### ENVI dimension validation

*Batch: 3.11.4*

The ENVI driver warns or errors when its samples, lines, or bands exceed `INT_MAX`, instead of accepting an unsupported dimension.

### Destination initialization warning semantics

*Batch: 3.11.5*

`InitializeDestinationBuffer()` no longer returns `CE_Failure` when `INIT_DEST=NO_DATA` is requested without a nodata value. It still warns and zero-initializes the destination buffer.

### LIBERTIFF RGB-to-RGBA reads

*Batch: 3.11.5*

LIBERTIFF correctly reads an RGB pixel-interleaved file into an RGBA pixel-interleaved buffer.

### Leap-second date parsing

*Batch: 3.11.5*

`OGRParseDate()` accepts timestamps containing leap seconds.

### Native-precision floating-point raster analysis

*Batch: 3.12.1*

`GDALFPolygonize()` now processes Float64 rasters at their native precision instead of converting values to Float32. `ComputeStatistics()` corrects Float64 standard deviations with SSE2/AVX2 and uses Float64 precision for Float32 mean and standard-deviation calculations.

### NITF extended-header TREs

*Batch: 3.12.1*

The NITF driver now reads TREs stored in the extended header correctly.

### RLE4-compressed BMP decoding

*Batch: 3.12.2*

The BMP driver now decodes RLE4-compressed images correctly.

### Negative HF2 elevations

*Batch: 3.12.2*

The HF2 driver now reads negative elevation values correctly.

### NITF RPFIMG coverage coordinates

*Batch: 3.12.2*

The NITF specification data corrects previously inverted latitude and longitude values in the RPFIMG `CoverageSectionSubheader`.

### Sentinel-2 geolocation with missing granules

*Batch: 3.12.2*

Geolocation-enabled Sentinel-2 subdatasets now tolerate expected missing granules.

### Restored ESRI WKT output

*Batch: 3.12.3*

`gdalinfo -wkt_format WKT1_ESRI` is supported again.

### Float16 GeoTIFF prediction

*Batch: 3.12.3*

The GTiff driver accepts `Float16` data with `PREDICTOR=3`. Creating a GeoTIFF also honors `GDAL_DISABLE_READDIR_ON_OPEN=TRUE` without listing the output directory.

### Driver connection and subdataset parsing

*Batch: 3.12.3*

GeoRaster preserves double quotes in database connection strings. `GDALGetSubdatasetInfo()` now handles netCDF subdataset names whose endpoint includes a port number.

### JP2 gray-plus-alpha channel definitions

*Batch: 3.12.4*

The JP2OpenJPEG writer no longer emits duplicate type/association pairs in the CDEF box for JPEG 2000 files containing three gray bands plus alpha.

### Raster blending, creation, and editing

*Batch: 3.13.0*

`gdal raster blend` adds multiply, screen, overlay, hard-light, darken, lighten, color-dodge, and color-burn modes. Raster creation can be a pipeline step and replicates `--like` tiling where possible, while raster editing can set color interpretation, scale, offset, and a color map or remove a color table.

### Raster sampling, selection, and matching

*Batch: 3.13.0*

`gdal raster pixel-info` can promote values to Z, take position datasets and layers, carry selected fields, write an output dataset, and run in a pipeline. Raster selection accepts color interpretations such as `red`, `alpha`, or `nir` and an `--exclude` mode, while raster reprojection adds `--like`.

### New raster data-source drivers

*Batch: 3.13.0*

New read-only drivers expose E57 two-dimensional images and CPHD data through the multidimensional API. JP2GROK provides JPEG 2000 reading and writing through the AGPLv3-licensed Grok toolkit.

### New random-write and product writers

*Batch: 3.13.0*

COG implements `GDALDriver::Create()` for random-write creation, MiraMonRaster gains creation, and S102 v3.0 plus S104/S111 v2.0 gain `CreateCopy()` writing. NITF adds CADRG writing, HEIF can write single-band images, and AVIF supports 16-bit encoding and decoding with libavif 1.4 or later.

### Raster format creation and open options

*Batch: 3.13.0*

NITF creation accepts `NOW` for `NITF_FDT` and `NITF_IDATIM`; MBTiles adds `ELEVATION_TYPE`; PDF adds `SAVE_DPI_TO_PAM`; and `gdal driver rpftoc create` builds CADRG A.TOC indexes. GTiff consumes ENVI sidecars for wavelength, FWHM, and bad-band metadata and reports `LAYOUT=COG` for structurally valid COGs even without GDAL's ghost area.

### Restored legacy drivers and ABI change

*Batch: 3.13.0*

The OGR Tiger and UK .NTF drivers are restored after their 3.11 removal, although both remain candidates for future removal. The shared library major version is bumped, so binary dependents must be rebuilt or use matching GDAL libraries.

### Lanczos validity-threshold removal

*Batch: 3.13.1*

Lanczos warping no longer applies a special case when fewer than half of the contributing source pixels are valid, so results around masks and nodata may differ from earlier releases.

### NaN conversion to signed integers

*Batch: 3.13.1*

The SSE2 `GDALCopyWords()` path now converts floating-point NaN values to zero for signed 8-, 16-, and 32-bit integer destinations, matching the scalar path.

### NITF wavelength units

*Batch: 3.13.1*

The NITF driver parses every `WAVE_LENGTH_UNIT` case in the `BANDSB` TRE.

### BigTIFF temporary files during COG creation

*Batch: 3.13.2*

`COGCreate()` always creates its temporary file as BigTIFF, so large COG creation is not constrained by a classic-TIFF intermediate.

### JP2Grok output-buffer handling

*Batch: 3.13.2*

JP2Grok handles `Float32`, `Float64`, and 16-bit output buffers and supports a genuinely single-threaded decode path.

### NITF CADRG compression identification

*Batch: 3.13.2*

The NITF driver accepts `IC=C4` when `PRODUCT_TYPE=CADRG`.
