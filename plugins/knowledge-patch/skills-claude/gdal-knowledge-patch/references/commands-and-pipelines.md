# Commands, algorithms, and pipelines

Use this reference for the task areas below. Batch labels identify when each behavior entered the covered compatibility history.

## Unified command family

### Restored raster API and CLI behavior

*Batch: 3.10.1*

`GDALContourGenerateEx()` returns `CE_None` for a constant-valued raster, reversing a 3.10.0 regression. `gdalinfo` again streams to standard output, and the `gdaltindex -ot` option removed accidentally in 3.10.0 is available again.

### Exact non-rectangular clipping in `ogr2ogr`

*Batch: 3.10.2*

`-clipsrc` and `-clipdst` now reject an input geometry that lies within a non-rectangular clip geometry's envelope but does not intersect the clip geometry itself.

### Unified `gdal` command family

*Batch: 3.11.0*

The new `gdal` front end groups operations into subcommands, including `gdal raster calc`, `gdal raster resclassify`, `gdal raster tile` (a C++ port of `gdal2tiles`), `gdal vsi list/copy/delete/move/sync`, and `gdal driver {driver_name}`. The algorithm framework is also exposed through C, C++, and Python APIs, and the command installs Bash completion.

### Unified overview controls

*Batch: 3.11.1*

`gdal raster overview add` accepts `-r none`. COG cleanup through `gdaladdo` or the unified overview commands exposes the `IGNORE_COG_LAYOUT_BREAK` message and automatically enables that option for `-clean`, which does not break the layout.

### Expanded unified raster operations

*Batch: 3.12.0*

The unified CLI adds `gdal raster as-features`, `blend`, `compare`, `neighbors`, `nodata-to-alpha`, `pansharpen`, `proximity`, `rgb-to-palette`, `update`, and `zonal-stats`. `fill-nodata`, `proximity`, `sieve`, and `viewshed` can be pipeline steps, while `mosaic` and `stack` can start a raster pipeline.

### Raster editing, clipping, and overview inputs

*Batch: 3.12.0*

`gdal raster clip` adds `--window <column>,<line>,<width>,<height>`, and `gdal raster edit` adds `--gcp` and `--unset-metadata-domain`. `gdal raster overview add` can take an existing overview with `--overview-src` and forwards overview creation settings through `--creation-option` or `--co`; `gdalbuildvrt` adds `-write_absolute_path`.

### Unified-source-nodata warping

*Batch: 3.12.2*

Warping with `UNIFIED_SRC_NODATA=YES` no longer applies inappropriate destination-nodata avoidance.

### Unified CLI option parity

*Batch: 3.12.3*

Pipeline-mode `gdal raster contour`, `gdal raster polygonize`, and `gdal vector select` expose `--output-layer`. Standalone `gdal raster edit` now exposes the `--oo` input open-option argument.

### Dataset copy and rename targets

*Batch: 3.12.3*

`gdal dataset copy` and `gdal dataset rename` now work with vector datasets and directories, rather than only their earlier target types.

### Expanded unified CLI

*Batch: 3.13.0*

New commands include `gdal vector combine`, `concave-hull`, `convex-hull`, `create`, `dissolve`, `export-schema`, `update`, `rename-layer`, and `sort`, plus `gdal dataset check` and COG and GeoPackage validation under `gdal driver`.

### Multidimensional CLI additions

*Batch: 3.13.0*

`gdal mdim info --summary` provides abbreviated output, and `gdal mdim mosaic` accepts dimensions that have no indexing variable.

### Curve layers in `gdal vector clip`

*Batch: 3.13.2*

`gdal vector clip` now works when the layer geometry type is a curve type.

## Pipelines and algorithm composition

### GDALG streamed vector pipelines

*Batch: 3.11.0*

The read-only GDALG (GDAL Streamed Algorithm Format) driver represents an on-the-fly vector dataset by replaying compatible `gdal` command lines, providing a VRT-like format for streamed algorithm pipelines.

### Composite and reusable pipelines

*Batch: 3.12.0*

`gdal pipeline` can mix raster and vector stages and supports nested pipelines plus a `tee` step. An existing pipeline can also be run while overriding or adding parameters.

### Raster inputs supplied through pipelines

*Batch: 3.12.1*

`gdal raster compare`, `info`, and `tile` work when a pipeline receives its input dataset outside the pipeline string. `gdal raster calc` also accepts input files represented by nested pipelines.

### Nested raster and vector pipelines

*Batch: 3.12.4*

Nested `gdal pipeline` definitions work when a stage such as vector concatenation can accept several input datasets. In raster pipelines, `gdal raster edit` can follow an anonymous VRT-producing stage without failing.

### Selected-layer vector SQL pipelines

*Batch: 3.12.4*

In `gdal vector pipeline`, `read --layer` now forwards `ExecuteSQL()` to the source dataset, so a selected-layer read can feed a subsequent `sql` step.

### External, multi-output, and append pipelines

*Batch: 3.13.0*

Pipelines gain an `external` step for running an external command, and the `_` placeholder dataset name can select a non-first dataset produced by the preceding step. Unified commands using `--append` now create the target dataset when it does not exist.

### Named pipeline materialization

*Batch: 3.13.1*

A pipeline `materialize` step with a named output now infers the output format; a sequence such as `... ! materialize --output my.tif ! tile` is supported.

### Tiled-pipeline standard output

*Batch: 3.13.1*

A `gdal raster pipeline ... ! tile` sequence no longer writes the output filename to standard output.

### Anonymous COG materialization before tiling

*Batch: 3.13.2*

A raster pipeline can materialize an unnamed COG and pass it directly to a tiling stage:

```text
gdal raster pipeline read byte.tif ! materialize --format COG ! tile
```

### OSM and PBF vector pipelines

*Batch: 3.13.2*

Vector pipelines that read an OSM or PBF source, perform an operation and filter, and then write the result now execute correctly.

## Utilities and automation contracts

### Double-valued `gdal_rasterize` target sizes

*Batch: 3.10.2*

`gdal_rasterize` now accepts double values for `-ts`:

```text
-ts 1024.0 512.0
```

### Zero and negative `gdaldem` azimuths

*Batch: 3.10.3*

`gdaldem` accepts zero or negative values for `-az`:

```text
gdaldem hillshade input.tif output.tif -az 0
```

### GeoPackage sources for `ogr2ogr -upsert`

*Batch: 3.10.3*

`ogr2ogr -upsert` now works when the source dataset is a GeoPackage.

### Algorithm argument and cancellation errors

*Batch: 3.11.1*

`GDALAlgorithm` rejects malformed list arguments and `NaN` values for range-constrained arguments. An interrupted `Run()` also reports `CE_Failure` through its progress function.

### Excluded-value controls in `gdal raster tile`

*Batch: 3.11.1*

The unified tiling command supports the `gdal2tiles` options `--excluded-values`, `--excluded-values-pct-threshold`, and `--nodata-values-pct-threshold`.

### JSON output contracts in `gdalinfo`

*Batch: 3.11.1*

JSON output represents an integer band's nodata value as an integer, attaches its raster attribute table as the band's `rat` object, and omits `wgs84Extent` and `extent` for non-georeferenced images.

### Bounds transformation for `gdalwarp`

*Batch: 3.11.1*

When `-te` and `-te_srs` are combined, `gdalwarp` computes the target-CRS extent with `OGRCoordinateTransformation::TransformBounds()`.

### Nodata queries with `gdallocationinfo`

*Batch: 3.11.2*

`gdallocationinfo` again handles nodata values correctly, fixing a regression introduced in 3.10.0.

### Larger `@filename` arguments

*Batch: 3.11.2*

`ogrinfo`, `ogr2ogr`, `gdal vector sql`, and related vector utilities accept `@filename` argument files up to 10 MB instead of 1 MB.

### Raster utility result semantics

*Batch: 3.11.4*

`gdal mdim info` returns status 0 on success. `gdal_footprint` reports failure when its single input feature cannot be simplified, and `gdal_viewshed` sets the DEM lower bound from the input raster.

### `gdaldem` on non-north-up rasters

*Batch: 3.11.5*

Aspect, TPI, and TRI results are corrected for non-north-up sources. Hillshade, slope, and roughness are also corrected for rotated sources.

### Expanded vector, multidimensional, and dataset operations

*Batch: 3.12.0*

New commands include `gdal vector check-coverage`, `check-geometry`, `clean-coverage`, `index`, `layer-algebra`, `make-point`, `partition`, `set-field-type`, and `simplify-coverage`, plus a vector-pipeline `limit` step. `gdal mdim mosaic` is new, and `gdal dataset` replaces the functionality of `gdal manage`.

### Raster calculation and composition controls

*Batch: 3.12.0*

`gdal raster calc` handles nodata, adds `--flatten`, and accepts `--dialect=muparser|builtin`; the built-in dialect can compute one output band from all bands of one input. `gdal raster mosaic` accepts `--pixel-function` and `--pixel-function-arg`, while `mosaic` and `stack` add `--absolute-path`.

### Correct STAC JSON from `gdalinfo`

*Batch: 3.12.1*

`gdalinfo -json` reports `stac:transform` coefficients in the correct order and sets `[stac][raster:bands][0][nodata]` for floating-point datasets.

### Vector SQL layer replacement

*Batch: 3.12.1*

`gdal vector sql --overwrite-layer` now performs the requested layer overwrite correctly.

### Windows spawn-mode raster tiling

*Batch: 3.12.1*

`gdal raster tile --parallel-mode=spawn` no longer stalls on Windows when `CPL_DEBUG=ON`.

### Target-aligned raster mosaics require a resolution

*Batch: 3.12.2*

`gdal raster mosaic` now checks that `--resolution` is supplied whenever `--target-aligned-pixels` is used.

### Raster calculations without geotransforms

*Batch: 3.12.3*

`gdal raster calc` now handles inputs that have no geotransform.

### Leading-space paths in `ogrmerge.py`

*Batch: 3.12.3*

`ogrmerge.py` now accepts input filenames that begin with spaces.

### Nodata filtering in `raster as-features`

*Batch: 3.12.4*

`gdal raster as-features --skip-nodata` no longer omits features that should remain in the output.

### Raster indexing and dataset discovery

*Batch: 3.13.0*

`gdal raster index` adds a `STAC-GeoParquet` profile, `filename`, `md5`, or `metadata-item` ID methods, and metadata-name and base-URL controls. `gdal dataset identify --detailed` can emit results through any writable vector driver, and text raster/vector info accepts `--crs-format=AUTO|WKT2|PROJJSON`.

### Dataset- and layer-option diagnostics

*Batch: 3.13.1*

Dataset creation warns when an unknown dataset creation option matches a layer creation option, and layer creation issues the converse warning.

### `ogrlineref` geometry inputs

*Batch: 3.13.2*

`ogrlineref` accepts a single-part `MULTILINESTRING` and handles non-line-string inputs without failing unsafely.

### `ogr2ogr` VRT error status

*Batch: 3.13.2*

`ogr2ogr` now returns a nonzero status when an error occurs while processing a VRT, allowing automation to detect the failure.

### Filters in `gdal vector limit`

*Batch: 3.13.2*

`gdal vector limit` now applies dataset filters instead of limiting an unfiltered stream.
