# Raster Processing

RasterIO, warping, resampling, nodata, masks, statistics, VRT composition, and raster algorithms.

## All-nodata contour inputs

*Batch 3.12.2*

Contouring an all-nodata raster now succeeds with an empty output layer instead of emitting an error.

## Arrow and directory-oriented vector capabilities

*Batch 3.13.0*

Arrow field creation and batch writing support string-view values, and the C API gains `OGR_L_GetAttributeFilter()`. A new driver capability identifies directories that may contain multiple vector layers and is advertised by Shapefile, MapInfo, CSV, FlatGeobuf, and MiraMonVector.

## BigTIFF nodata values in LIBERTIFF

*Batch 3.13.1*

LIBERTIFF correctly reads a BigTIFF nodata value whose string representation occupies four through eight bytes.

## COG and TIFF creation behavior

*Batch 3.11.0*

COG creation supports `INTERLEAVE=BAND` and `TILE`, notably for hyperspectral data. GTiff reads ArcGIS-style `.tif.vat.dbf` raster attribute tables, and GTiff, COG, and warping preserve premultiplied-alpha information from source TIFFs.

## Complete VRT overview exposure

*Batch 3.11.2*

A single-source VRT exposes all source overviews regardless of their size. `VRTPansharpen` also tolerates source bands with differing numbers of overviews when generating virtual overviews.

## Complex COG and RGB-NIR GeoTIFF creation

*Batch 3.11.4*

The COG driver can create datasets with complex data types. The GTiff driver can create R, G, B, NIR files without an explicit `PHOTOMETRIC` creation option.

## Degenerate line geometries in MIF

*Batch 3.12.2*

The MITAB `.mif` reader now accepts line strings and multi-line strings containing one point or no points.

## Destination initialization warning semantics

*Batch 3.11.5*

`InitializeDestinationBuffer()` no longer returns `CE_Failure` when `INIT_DEST=NO_DATA` is requested without a nodata value. It still warns and zero-initializes the destination buffer.

## Empty GML curves

*Batch 3.11.1*

The GML geometry parser interprets `<gml:Curve><gml:segments/></gml:Curve>` as `LINESTRING EMPTY`.

## Exact integer nodata statistics

*Batch 3.13.2*

`ComputeRasterMinMax()` and `GetHistogram()` now require an exact integer match when excluding a nodata value, changing results in cases that previously treated a different integer as nodata.

## FlatGeobuf output without a spatial index

*Batch 3.10.1*

With `SPATIAL_INDEX=NO`, the FlatGeobuf writer accepts a dataset with no features and handles empty geometries as null geometries.

## Geodesic lengths for open line strings

*Batch 3.10.2*

`GeodesicLength()` works on non-closed line strings again; a regression in 3.10.0 had limited it to closed line strings.

## GML 3D geometry discovery

*Batch 3.12.1*

The GML reader accepts 3D geometries whose `srsName` is three-dimensional even without `srsDimension='3'`. When several geometry elements exist and the last one is consistently selected, that geometry column now receives a name.

## GTI relative paths and masked overview reads

*Batch 3.12.4*

Relative filenames in GTI XML or `.gti.gpkg` indexes are resolved relative to the main file. Downsampled requests on a GTI dataset with a mask band and overviews no longer fail with a `panBandMap[0]` missing-band error.

## GTI unreadable-source failures

*Batch 3.11.5*

A GTI raster read now fails when one of its sources is unreadable instead of allowing the failed source read to pass unnoticed.

## Implicit VRT derived-band overviews

*Batch 3.12.4*

`VRTDerivedRasterBand` now creates implicit overviews correctly.

## ISO-compliant GML center-point circles

*Batch 3.10.2*

`gml:CircleByCenterPoint()` now returns a five-point `CIRCULARSTRING`, complying with ISO/IEC 13249-3:2011.

## JSON-FG and Parquet evolution

*Batch 3.12.0*

JSON-FG is updated to specification 0.3.0 with read/write support for curve and measured geometries. Parquet gains editable-layer update support, reads and writes the Parquet `GEOMETRY` type with libarrow 21 or later, and adds a `COMPRESSION_LEVEL` layer creation option.

## Lanczos validity-threshold removal

*Batch 3.13.1*

Lanczos warping no longer applies a special case when fewer than half of the contributing source pixels are valid, so results around masks and nodata may differ from earlier releases.

## Large, global, and TPS warps

*Batch 3.11.4*

Warping large rasters no longer fails in cases such as globally extensive WMTS inputs, and a whole longitude range of at least 360 degrees is no longer given an inappropriate `CENTER_LONG` when targeting Web Mercator. TPS warping now defaults to `-wo SOURCE_EXTRA=5`.

## LIBKML creation types

*Batch 3.11.1*

The LIBKML driver advertises Date, Time, DateTime, and Integer64 fields during creation, mapping them to strings, and maps boolean fields correctly.

## LIBKML field-name collisions

*Batch 3.12.2*

When a LIBKML simple field has the same name as a core attribute, the driver appends `2` to the simple field's name.

## Masked naked Lerc2 files

*Batch 3.13.1*

The MRF driver can decode naked Lerc2 files containing masks when built with liblerc 3.0 or newer.

## More general OGR VRT source regions

*Batch 3.10.1*

An OGR VRT `SrcRegion` accepts any geometry type, as does `SetSpatialFilter()`, and `SrcRegion.clip` is applied correctly at the `OGRVRTLayer` level.

## Multipolygon results from edge-built polygons

*Batch 3.11.5*

`OGRBuildPolygonFromEdges()` returns a multipolygon when the edges require one, including for affected DXF `HATCH` geometries. Callers must therefore be prepared for a multipolygon result.

## Multithreaded warp interruption

*Batch 3.12.4*

Multithreaded warps detect progress interruption more reliably, and warping initiated from a worker thread avoids a potential deadlock.

## Native-precision floating-point raster analysis

*Batch 3.12.1*

`GDALFPolygonize()` now processes Float64 rasters at their native precision instead of converting values to Float32. `ComputeStatistics()` corrects Float64 standard deviations with SSE2/AVX2 and uses Float64 precision for Float32 mean and standard-deviation calculations.

## Nodata on pansharpened VRT overviews

*Batch 3.11.5*

`VRTPansharpenedRasterBand` overview bands now inherit the nodata value of the full-resolution band.

## Pansharpened VRT serialization and orientation

*Batch 3.12.2*

Pansharpened VRTs now serialize correctly when the panchromatic and multispectral bands have different extents. The vertical-orientation test for input datasets is also corrected.

## Pansharpening nearly aligned inputs

*Batch 3.10.3*

Pansharpening no longer reports I/O errors when the extents of the panchromatic and multispectral bands differ by less than one multispectral-band resolution.

## Parquet list-field handling

*Batch 3.12.1*

The Parquet driver adds the `LISTS_AS_STRING_JSON=YES/NO` open option. `SetIgnoredFields()` also works for fields whose type is a list of structures.

## PostgreSQL string truncation restored

*Batch 3.11.3*

The PostgreSQL driver again truncates strings as intended, restoring behavior that was broken in 3.11.1.

## Raster band algebra API

*Batch 3.12.0*

The C, C++, and Python APIs support arithmetic and comparison directly on raster bands, type conversion with `AsType()`, and algebra functions including `abs()`, `sqrt()`, logarithms, `min()`, `max()`, `mean()`, and `IfThenElse()`.

## Raster reads with masks, NaNs, and constant histograms

*Batch 3.11.4*

`GDALNoDataMaskBand::IRasterIO()` no longer corrupts Byte-band reads when `nLineSpace > nBufXSize`. Overview mode resampling accounts for `NaN` in `Float16` and `CFloat16`, while `GetDefaultHistogram()` handles constant-valued non-Byte data where `min == max`.

## Repeated GMLAS string-list elements

*Batch 3.10.3*

The GMLAS driver now reads every value when a `StringList` field is represented by a repeated element.

## Resampling with NaN nodata

*Batch 3.12.4*

Bilinear, cubic, cubic-spline, and Lanczos resampling now handle NaN values correctly when the band's nodata value is also NaN.

## Richer VRT composition

*Batch 3.11.0*

VRT pixel functions can evaluate arbitrary expressions, reclassify values, and apply `mul` or `sum` with a constant factor to one band. A `<SimpleSource>` or `<ComplexSource>` may embed a `<VRTDataset>` instead of naming a source file, and processed VRTs gain an `OutputBands` element for declaring output count and data types.

## RMS overview normalization

*Batch 3.12.3*

RMS overview resampling uses a corrected normalization formula, changing values produced by affected overviews.

## S-102 products without uncertainty

*Batch 3.11.1*

The S102 driver opens products that have no uncertainty component and retrieves nodata correctly when only a depth component is present.

## SQLite 3.49.1 with double-quoted strings disabled

*Batch 3.10.3*

The SQLite SQL dialect and the GeoPackage driver now work with SQLite 3.49.1 built or configured with `SQLITE_DQS=0`.

## Strided VRT derived-band reads

*Batch 3.12.3*

`VRTDerivedRasterBand::IRasterIO()` correctly zero-initializes output buffers when line spacing differs from pixel spacing multiplied by the buffer width.

## Unified-source-nodata warping

*Batch 3.12.2*

Warping with `UNIFIED_SRC_NODATA=YES` no longer applies inappropriate destination-nodata avoidance.

## VRT derived functions and block selection

*Batch 3.13.0*

VRT derived bands add `area`, `quantile`, and `round` pixel functions. The `vrt://` connection protocol also accepts a `block` option.

## VRT derived-band functions and expressions

*Batch 3.12.0*

VRT pixel functions add `mean`, `median`, `geometric_mean`, `harmonic_mean`, `mode`, `argmin`, and `argmax`, and now account for nodata; `min` and `max` accept an optional `k` constant. Muparser expressions add `fmod`, derived-band expressions expose `_CENTER_X_` and `_CENTER_Y_`, and `vrt://` accepts a `transpose` option.

## VRT processed-dataset scaling

*Batch 3.10.1*

Processed VRT datasets now read scale and offset from their source dataset.

## VRT source schema and deterministic reads

*Batch 3.12.1*

The VRT XML schema permits a `name` attribute on source types. Nearest-neighbor reads use the generic raster-band coordinate rounding, and multithreading is disabled for neighboring sources that are not aligned to an integer output coordinate so pixel output remains deterministic.

## Warping empty source windows

*Batch 3.10.3*

Warping with the MEM driver now handles an empty source window correctly when the nodata value is nonzero.

## WEBP-compressed RGBA in LIBERTIFF

*Batch 3.11.2*

LIBERTIFF reads WEBP-compressed RGBA images even when a fully opaque tile or strip omits its alpha component.
