# CLI and Pipelines

Unified commands, pipelines, utility behavior, argument handling, and machine-readable output.

## Algorithm argument and cancellation errors

*Batch 3.11.1*

`GDALAlgorithm` rejects malformed list arguments and `NaN` values for range-constrained arguments. An interrupted `Run()` also reports `CE_Failure` through its progress function.

## Anonymous COG materialization before tiling

*Batch 3.13.2*

A raster pipeline can materialize an unnamed COG and pass it directly to a tiling stage:

```text
gdal raster pipeline read byte.tif ! materialize --format COG ! tile
```

## Arrow-backed field selection

*Batch 3.12.1*

`ogr2ogr` and `VectorTranslate` correctly apply `selectFields` through the Arrow code path.

## Automatic source-overview selection for tiling

*Batch 3.13.1*

`gdal raster tile` now selects an appropriate source overview automatically.

## Bounds transformation for `gdalwarp`

*Batch 3.11.1*

When `-te` and `-te_srs` are combined, `gdalwarp` computes the target-CRS extent with `OGRCoordinateTransformation::TransformBounds()`.

## COG translation with a selected mask band

*Batch 3.11.5*

`gdal_translate -of COG -b 1 -b 2 -b 3 -b mask ...` can translate an RGB dataset with overviews without crashing; the selected mask becomes a regular output band tagged as alpha.

## `ComplexSource` calculation types

*Batch 3.12.1*

`gdal raster calc` and `VRTDerivedRasterBand` now use the correct computation and transfer data types with a `ComplexSource`.

## Composite and reusable pipelines

*Batch 3.12.0*

`gdal pipeline` can mix raster and vector stages and supports nested pipelines plus a `tee` step. An existing pipeline can also be run while overriding or adding parameters.

## Correct STAC JSON from `gdalinfo`

*Batch 3.12.1*

`gdalinfo -json` reports `stac:transform` coefficients in the correct order and sets `[stac][raster:bands][0][nodata]` for floating-point datasets.

## Current GMT color tables

*Batch 3.13.1*

`gdal raster color-map` accepts current GMT `.cpt` color-table files.

## Curve layers in `gdal vector clip`

*Batch 3.13.2*

`gdal vector clip` now works when the layer geometry type is a curve type.

## Dataset copy and rename targets

*Batch 3.12.3*

`gdal dataset copy` and `gdal dataset rename` now work with vector datasets and directories, rather than only their earlier target types.

## Double-valued `gdal_rasterize` target sizes

*Batch 3.10.2*

`gdal_rasterize` now accepts double values for `-ts`:

```text
-ts 1024.0 512.0
```

## Exact non-rectangular clipping in `ogr2ogr`

*Batch 3.10.2*

`-clipsrc` and `-clipdst` now reject an input geometry that lies within a non-rectangular clip geometry's envelope but does not intersect the clip geometry itself.

## Excluded-value controls in `gdal raster tile`

*Batch 3.11.1*

The unified tiling command supports the `gdal2tiles` options `--excluded-values`, `--excluded-values-pct-threshold`, and `--nodata-values-pct-threshold`.

## Expanded unified CLI

*Batch 3.13.0*

New commands include `gdal vector combine`, `concave-hull`, `convex-hull`, `create`, `dissolve`, `export-schema`, `update`, `rename-layer`, and `sort`, plus `gdal dataset check` and COG and GeoPackage validation under `gdal driver`.

## Expanded unified raster operations

*Batch 3.12.0*

The unified CLI adds `gdal raster as-features`, `blend`, `compare`, `neighbors`, `nodata-to-alpha`, `pansharpen`, `proximity`, `rgb-to-palette`, `update`, and `zonal-stats`. `fill-nodata`, `proximity`, `sieve`, and `viewshed` can be pipeline steps, while `mosaic` and `stack` can start a raster pipeline.

## Expanded vector, multidimensional, and dataset operations

*Batch 3.12.0*

New commands include `gdal vector check-coverage`, `check-geometry`, `clean-coverage`, `index`, `layer-algebra`, `make-point`, `partition`, `set-field-type`, and `simplify-coverage`, plus a vector-pipeline `limit` step. `gdal mdim mosaic` is new, and `gdal dataset` replaces the functionality of `gdal manage`.

## External, multi-output, and append pipelines

*Batch 3.13.0*

Pipelines gain an `external` step for running an external command, and the `_` placeholder dataset name can select a non-first dataset produced by the preceding step. Unified commands using `--append` now create the target dataset when it does not exist.

## Filters in `gdal vector limit`

*Batch 3.13.2*

`gdal vector limit` now applies dataset filters instead of limiting an unfiltered stream.

## `gdaldem` on non-north-up rasters

*Batch 3.11.5*

Aspect, TPI, and TRI results are corrected for non-north-up sources. Hillshade, slope, and roughness are also corrected for rotated sources.

## GDALG streamed vector pipelines

*Batch 3.11.0*

The read-only GDALG (GDAL Streamed Algorithm Format) driver represents an on-the-fly vector dataset by replaying compatible `gdal` command lines, providing a VRT-like format for streamed algorithm pipelines.

## Geometry-check result fields

*Batch 3.12.1*

`gdal vector check-geometry` adds the `--include-field` option for including a field in its results.

## GeoPackage sources for `ogr2ogr -upsert`

*Batch 3.10.3*

`ogr2ogr -upsert` now works when the source dataset is a GeoPackage.

## JPEG XL conversion and lossy-option diagnostics

*Batch 3.11.4*

`gdal_translate non_byte.jxl byte.jxl -ot Byte` now converts JPEG XL data correctly. GTiff and COG emit warnings when `JXL_DISTANCE` or `JXL_ALPHA_DISTANCE` is used without `JXL_LOSSLESS=NO`.

## JSON output contracts in `gdalinfo`

*Batch 3.11.1*

JSON output represents an integer band's nodata value as an integer, attaches its raster attribute table as the band's `rat` object, and omits `wgs84Extent` and `extent` for non-georeferenced images.

## Larger `@filename` arguments

*Batch 3.11.2*

`ogrinfo`, `ogr2ogr`, `gdal vector sql`, and related vector utilities accept `@filename` argument files up to 10 MB instead of 1 MB.

## Larger vector concatenations

*Batch 3.11.4*

`gdal vector concat` accepts more than 1,000 input files.

## Leading-space paths in `ogrmerge.py`

*Batch 3.12.3*

`ogrmerge.py` now accepts input filenames that begin with spaces.

## Multidimensional CLI additions

*Batch 3.13.0*

`gdal mdim info --summary` provides abbreviated output, and `gdal mdim mosaic` accepts dimensions that have no indexing variable.

## Multivalue multidimensional conversion

*Batch 3.12.1*

`gdal mdim convert` now accepts multiple values for `--group`, `--subset`, and `--scale-axes`.

## Named pipeline materialization

*Batch 3.13.1*

A pipeline `materialize` step with a named output now infers the output format; a sequence such as `... ! materialize --output my.tif ! tile` is supported.

## Nested raster and vector pipelines

*Batch 3.12.4*

Nested `gdal pipeline` definitions work when a stage such as vector concatenation can accept several input datasets. In raster pipelines, `gdal raster edit` can follow an anonymous VRT-producing stage without failing.

## Nodata filtering in `raster as-features`

*Batch 3.12.4*

`gdal raster as-features --skip-nodata` no longer omits features that should remain in the output.

## Nodata queries with `gdallocationinfo`

*Batch 3.11.2*

`gdallocationinfo` again handles nodata values correctly, fixing a regression introduced in 3.10.0.

## OCI time-zone timestamp creation

*Batch 3.10.1*

The OCI driver adds a `TIMESTAMP_WITH_TIME_ZONE` layer creation option, with corresponding `ogr2ogr` handling.

## `ogr2ogr` VRT error status

*Batch 3.13.2*

`ogr2ogr` now returns a nonzero status when an error occurs while processing a VRT, allowing automation to detect the failure.

## `ogrlineref` geometry inputs

*Batch 3.13.2*

`ogrlineref` accepts a single-part `MULTILINESTRING` and handles non-line-string inputs without failing unsafely.

## OSM and PBF vector pipelines

*Batch 3.13.2*

Vector pipelines that read an OSM or PBF source, perform an operation and filter, and then write the result now execute correctly.

## Other upgrade compatibility changes

*Batch 3.11.0*

The OpenCL warper and the unofficial `gdalwarpsimple` and `ogrdissolve` applications were removed; the OGR `Memory` driver is deprecated and aliases the unified `MEM` driver, and the shared-library major version was bumped. FileGDB update and creation now route through OpenFileGDB, while PDF creation no longer supports `GEO_ENCODING=OGC_BP`.

## Out-of-range nodata warnings in separate VRTs

*Batch 3.13.1*

`gdalbuildvrt -separate` warns when a nodata value is outside the range of the target band type.

## Palette, zonal, and rasterization controls

*Batch 3.13.0*

`gdal raster rgb-to-palette` adds `--output-nodata`, `--no-dither`, and `--bit-depth`; zonal statistics accepts `--include-field ALL|NONE`, `--include-geom`, and an output layer. Rasterization can derive one output size from the other size and the input extent when one size is zero.

## Partitioned vector datasets

*Batch 3.13.0*

`gdal vector partition` can partition by geometry type, makes `--field` optional, and automatically creates the Parquet `_metadata` index. The same index can be built independently with `gdal driver parquet create-metadata-file`.

## Polar-to-geographic reprojection

*Batch 3.11.5*

Geometry reprojection from a polar CRS to geographic coordinates is corrected in both the core transformation path and `gdal vector reproject`.

## Python algorithm namespace

*Batch 3.12.0*

Python exposes the algorithm registry through a dynamically generated `gdal.alg` module:

```python
gdal.alg.raster.convert(input="in.tif", output="out.tif")
```

## Raster blending, creation, and editing

*Batch 3.13.0*

`gdal raster blend` adds multiply, screen, overlay, hard-light, darken, lighten, color-dodge, and color-burn modes. Raster creation can be a pipeline step and replicates `--like` tiling where possible, while raster editing can set color interpretation, scale, offset, and a color map or remove a color table.

## Raster calculation and composition controls

*Batch 3.12.0*

`gdal raster calc` handles nodata, adds `--flatten`, and accepts `--dialect=muparser|builtin`; the built-in dialect can compute one output band from all bands of one input. `gdal raster mosaic` accepts `--pixel-function` and `--pixel-function-arg`, while `mosaic` and `stack` add `--absolute-path`.

## Raster calculations without geotransforms

*Batch 3.12.3*

`gdal raster calc` now handles inputs that have no geotransform.

## Raster editing, clipping, and overview inputs

*Batch 3.12.0*

`gdal raster clip` adds `--window <column>,<line>,<width>,<height>`, and `gdal raster edit` adds `--gcp` and `--unset-metadata-domain`. `gdal raster overview add` can take an existing overview with `--overview-src` and forwards overview creation settings through `--creation-option` or `--co`; `gdalbuildvrt` adds `-write_absolute_path`.

## Raster format creation and open options

*Batch 3.13.0*

NITF creation accepts `NOW` for `NITF_FDT` and `NITF_IDATIM`; MBTiles adds `ELEVATION_TYPE`; PDF adds `SAVE_DPI_TO_PAM`; and `gdal driver rpftoc create` builds CADRG A.TOC indexes. GTiff consumes ENVI sidecars for wavelength, FWHM, and bad-band metadata and reports `LAYOUT=COG` for structurally valid COGs even without GDAL's ghost area.

## Raster indexing and dataset discovery

*Batch 3.13.0*

`gdal raster index` adds a `STAC-GeoParquet` profile, `filename`, `md5`, or `metadata-item` ID methods, and metadata-name and base-URL controls. `gdal dataset identify --detailed` can emit results through any writable vector driver, and text raster/vector info accepts `--crs-format=AUTO|WKT2|PROJJSON`.

## Raster inputs supplied through pipelines

*Batch 3.12.1*

`gdal raster compare`, `info`, and `tile` work when a pipeline receives its input dataset outside the pipeline string. `gdal raster calc` also accepts input files represented by nested pipelines.

## Raster sampling, selection, and matching

*Batch 3.13.0*

`gdal raster pixel-info` can promote values to Z, take position datasets and layers, carry selected fields, write an output dataset, and run in a pipeline. Raster selection accepts color interpretations such as `red`, `alpha`, or `nir` and an `--exclude` mode, while raster reprojection adds `--like`.

## Raster utility options and stricter behavior

*Batch 3.11.0*

`gdalbuildvrt` adds `-co` and `-resolution same|compatible`; `gdaldem` derives scale from the CRS and adds `-xscale`/`-yscale`; `gdallocationinfo` can query corners; `rgb2pct` adds `--creation-option`; `gdal2xyz` can write to VSI paths; and `gdalenhance` is now installed and documented. `gdal_translate -projwin` includes partially covered pixels and transforms the full bounds, translation and warping reject invalid numeric options, nodata is copied only when exactly representable, polygonized contours omit min/max fields, and `gdal2tiles` applies source nodata even without reprojection.

## Raster utility result semantics

*Batch 3.11.4*

`gdal mdim info` returns status 0 on success. `gdal_footprint` reports failure when its single input feature cannot be simplified, and `gdal_viewshed` sets the DEM lower bound from the input raster.

## Reprojected and non-square-pixel extents

*Batch 3.12.3*

`gdaltindex` now uses GDALWarp when computing reprojected extents. `gdal2tiles` also computes the correct extent for source rasters with non-square pixels.

## Reprojecting directly to COG

*Batch 3.11.2*

`gdalwarp` can again reproject into Cloud Optimized GeoTIFF output, fixing a regression introduced in 3.11.0.

## Reprojection, resizing, tiling, and viewshed controls

*Batch 3.12.0*

`gdal raster reproject` adds `-j`/`--num-threads` and defaults to `ALL_CPUS`, while `gdal raster resize` adds `--resolution`. Raster tiling supports `--parallel-method=fork` on non-Windows systems or `spawn`, emits `stacta.json`, and can terminate a pipeline; viewshed adds angular, pitch, and minimum-distance masking.

## Restored ESRI WKT output

*Batch 3.12.3*

`gdalinfo -wkt_format WKT1_ESRI` is supported again.

## Restored raster API and CLI behavior

*Batch 3.10.1*

`GDALContourGenerateEx()` returns `CE_None` for a constant-valued raster, reversing a 3.10.0 regression. `gdalinfo` again streams to standard output, and the `gdaltindex -ot` option removed accidentally in 3.10.0 is available again.

## Selected-layer vector SQL pipelines

*Batch 3.12.4*

In `gdal vector pipeline`, `read --layer` now forwards `ExecuteSQL()` to the source dataset, so a selected-layer read can feed a subsequent `sql` step.

## Single-file VSI timestamps

*Batch 3.13.1*

`gdal vsi list` displays the last-modification timestamp when its target is a single file.

## Stricter vector conversion failures

*Batch 3.13.0*

`ogr2ogr` now fails by default when destination field creation fails unless `-skip` is supplied. It and `gdal vector convert` also warn when the output driver cannot preserve curve, Z, or M geometries.

## Sum-resampled warps

*Batch 3.12.1*

`gdalwarp -r sum` no longer introduces artifacts related to chunked processing.

## Target-aligned raster mosaics require a resolution

*Batch 3.12.2*

`gdal raster mosaic` now checks that `--resolution` is supplied whenever `--target-aligned-pixels` is used.

## Three-dimensional geometry validation

*Batch 3.12.1*

`gdal vector make-valid` no longer skips 3D geometries.

## Tiled-pipeline standard output

*Batch 3.13.1*

A `gdal raster pipeline ... ! tile` sequence no longer writes the output filename to standard output.

## Timestamp-based GTI overview refresh

*Batch 3.11.1*

`gdaladdo --partial-refresh-from-source-timestamp` works with GTI datasets as well as VRT datasets.

## Unified CLI option parity

*Batch 3.12.3*

Pipeline-mode `gdal raster contour`, `gdal raster polygonize`, and `gdal vector select` expose `--output-layer`. Standalone `gdal raster edit` now exposes the `--oo` input open-option argument.

## Unified `gdal` command family

*Batch 3.11.0*

The new `gdal` front end groups operations into subcommands, including `gdal raster calc`, `gdal raster resclassify`, `gdal raster tile` (a C++ port of `gdal2tiles`), `gdal vsi list/copy/delete/move/sync`, and `gdal driver {driver_name}`. The algorithm framework is also exposed through C, C++, and Python APIs, and the command installs Bash completion.

## Unified overview controls

*Batch 3.11.1*

`gdal raster overview add` accepts `-r none`. COG cleanup through `gdaladdo` or the unified overview commands exposes the `IGNORE_COG_LAYOUT_BREAK` message and automatically enables that option for `-clean`, which does not break the layout.

## Valid bundled JSON schemas

*Batch 3.12.4*

JSON Schema validation errors in the bundled `gdalinfo` and `ogrinfo` schemas are fixed.

## Vector SQL layer replacement

*Batch 3.12.1*

`gdal vector sql --overwrite-layer` now performs the requested layer overwrite correctly.

## Vector update and inspection workflows

*Batch 3.12.0*

`gdal vector convert` can update an existing output and accept output open options, and `gdal vector write`/`convert` add `--upsert`; `gdal vector sql --update` performs in-place changes. Conversion no longer assumes Shapefile when the output has no `.shp` extension, and text-mode `gdal vector info` needs `--features` to emit features and accepts `--limit`.

## Vector workflow preservation and inspection

*Batch 3.13.0*

Unified vector algorithms propagate dataset field domains, relationships, and metadata. Vector filtering adds `--update-extent`, vector info adds `--fid`, and pipelines can use `--no-create-empty-layers`.

## Vertical-shift unit metadata

*Batch 3.13.1*

For a 3D-to-3D vertical-shift warp, `gdalwarp` no longer copies the source unit type to the output file.

## Warp and coordinate-operation controls

*Batch 3.11.0*

The transformer adds a Homography type, while the warper adds `MODE_TIES`, uses source-pixel coverage for mode resampling, and permits a mode of `-1`. Transformer options now include `ALLOW_BALLPARK=NO`, `ONLY_BEST=AUTO/YES/NO`, source/destination axis-mapping controls, and `HEIGHT_DEFAULT` as the fallback RPC height; `ogr2ogr -ct_opt` exposes the ballpark, best-operation, and differing-operation-warning controls.

## Warp data types

*Batch 3.12.3*

`GDALWarpResolveWorkingDataType()` now examines band data types before falling back to `UInt8`, and nearest-neighbor warping has a dedicated `Int8` path. Signed-byte inputs therefore no longer depend on byte-oriented working-type behavior.

## Warp destination initialization

*Batch 3.13.0*

`gdalwarp` now fails when `INIT_DEST=NO_DATA` is requested without a nodata value. The new `RESET_DEST_PIXELS=YES|NO` warp option can completely reset an existing destination to destination nodata or zero.

## Windows spawn-mode raster tiling

*Batch 3.12.1*

`gdal raster tile --parallel-mode=spawn` no longer stalls on Windows when `CPL_DEBUG=ON`.

## Zero and negative `gdaldem` azimuths

*Batch 3.10.3*

`gdaldem` accepts zero or negative values for `-az`:

```text
gdaldem hillshade input.tif output.tif -az 0
```

## Zonal statistics at and beyond raster bounds

*Batch 3.12.1*

`GDALZonalStats` handles affected polygons outside the raster extent, while `gdal raster zonal-stats` avoids integer overflow for geometries with huge coordinate values.
