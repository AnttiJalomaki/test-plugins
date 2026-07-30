# Raster Formats

Raster driver creation, decoding, metadata, validation, and format-specific interoperability.

## Additional imagery-format capabilities

*Batch 3.11.0*

HEIF gains tile reading, `CreateCopy()`, and read-only GeoHEIF support with libheif 1.19; AVIF gains read-only GeoHEIF support with the libavif development version current at release time, and JPEGXL reads Float16 as Float32. DIMAP exposes PNEO FWHM and RPC `HEIGHT_DEFAULT`, NITF represents SAR I/Q pairs as one complex band, Sentinel-2 recognizes `S2C_` names, and Leveller accepts document versions through 12.

## AVIF images larger than 10 MB

*Batch 3.10.3*

The AVIF driver can now read images larger than 10 MB.

## BigTIFF temporary files during COG creation

*Batch 3.13.2*

`COGCreate()` always creates its temporary file as BigTIFF, so large COG creation is not constrained by a classic-TIFF intermediate.

## DIMAP2 coverage metadata

*Batch 3.13.2*

The DIMAP2 driver reports `CLOUD_COVERAGE` and `SNOW_COVERAGE` metadata items.

## ENVI dimension validation

*Batch 3.11.4*

The ENVI driver warns or errors when its samples, lines, or bands exceed `INT_MAX`, instead of accepting an unsupported dimension.

## FLIR thermal JPEG handling

*Batch 3.11.1*

The JPEG driver reads FLIR thermal images stored as little-endian 16-bit PNG data. It exposes `IRWindowTransmission` separately instead of overwriting `IRWindowTemperature`, and corrects the metadata subdomain for `RelativeHumidity`.

## Float16 GeoTIFF prediction

*Batch 3.12.3*

The GTiff driver accepts `Float16` data with `PREDICTOR=3`. Creating a GeoTIFF also honors `GDAL_DISABLE_READDIR_ON_OPEN=TRUE` without listing the output directory.

## GRIB2 Transverse Mercator variants

*Batch 3.10.3*

The GRIB2 driver now reads Transverse Mercator definitions with negative easting/falsing values or a scale factor other than `0.9996`.

## IIIF Image API 3.0

*Batch 3.11.1*

The WMS driver adds a mini-driver for International Image Interoperability Framework Image API 3.0.

## Immediate reads of compressed multithreaded GeoTIFF output

*Batch 3.10.3*

A compressed GeoTIFF created in multithreaded mode can now be read immediately after creation, fixing a regression introduced in 3.10.1.

## JP2 gray-plus-alpha channel definitions

*Batch 3.12.4*

The JP2OpenJPEG writer no longer emits duplicate type/association pairs in the CDEF box for JPEG 2000 files containing three gray bands plus alpha.

## JP2Grok output-buffer handling

*Batch 3.13.2*

JP2Grok handles `Float32`, `Float64`, and 16-bit output buffers and supports a genuinely single-threaded decode path.

## JPEG XL from DNG in GeoTIFF

*Batch 3.10.1*

The GTiff driver supports `Float16` and TIFF compression value `52546`, the JPEG XL encoding defined by DNG 1.7.

## LIBERTIFF RGB-to-RGBA reads

*Batch 3.11.5*

LIBERTIFF correctly reads an RGB pixel-interleaved file into an RGBA pixel-interleaved buffer.

## MRF cache configuration rename

*Batch 3.13.2*

The MRF caching configuration replaces `MRF_BYPASSCACHING` with `MRF_ENABLE_CACHING`; deployments setting the former variable must migrate to the latter.

## Negative HF2 elevations

*Batch 3.12.2*

The HF2 driver now reads negative elevation values correctly.

## New random-write and product writers

*Batch 3.13.0*

COG implements `GDALDriver::Create()` for random-write creation, MiraMonRaster gains creation, and S102 v3.0 plus S104/S111 v2.0 gain `CreateCopy()` writing. NITF adds CADRG writing, HEIF can write single-band images, and AVIF supports 16-bit encoding and decoding with libavif 1.4 or later.

## NITF CADRG compression identification

*Batch 3.13.2*

The NITF driver accepts `IC=C4` when `PRODUCT_TYPE=CADRG`.

## NITF extended-header TREs

*Batch 3.12.1*

The NITF driver now reads TREs stored in the extended header correctly.

## NITF RPFIMG coverage coordinates

*Batch 3.12.2*

The NITF specification data corrects previously inverted latitude and longitude values in the RPFIMG `CoverageSectionSubheader`.

## NITF wavelength units

*Batch 3.13.1*

The NITF driver parses every `WAVE_LENGTH_UNIT` case in the `BANDSB` TRE.

## PNG caching without a band-one read

*Batch 3.11.2*

The PNG driver correctly caches other bands even when reading does not begin with band 1.

## Restored BT raster support

*Batch 3.11.4*

The BT driver is available again after its removal in 3.11.0.

## Restored GSAG raster support

*Batch 3.11.2*

The GSAG driver for Golden Software ASCII Grid is available again after its removal in 3.11.0.

## Restored GSBG raster support

*Batch 3.11.1*

The GSBG driver for Golden Software Surfer Binary Grid 6.0 is restored after its removal in 3.11.0.

## RLE4-compressed BMP decoding

*Batch 3.12.2*

The BMP driver now decodes RLE4-compressed images correctly.

## STAC 1.1 and STACIT identification

*Batch 3.10.2*

The STACIT driver supports STAC 1.1. Its `Identify()` method accepts an item when at least two of `proj:transform`, `proj:bbox`, and `proj:shape` are present.

## STACIT pagination requests

*Batch 3.12.1*

The STACIT driver no longer sends an initial pagination request with an empty `{}` body, improving compatibility with services that reject such a body.

## WEBP-compressed MBTiles updates

*Batch 3.10.3*

The MBTiles driver can now update datasets that use WEBP compression.
