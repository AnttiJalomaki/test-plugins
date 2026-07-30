# Text, Internationalization, and Media

Use this reference for PCRE, mbstring, Intl and locale behavior, formatting,
GD, EXIF, HEIF/HEIC, SVG, and image dimensions.

## Validation and text indexing

### Stricter numeric and domain validation (8.4-migration)

Invalid ranges raise `ValueError` in APIs including `curl_multi_select()`, GD
quality, speed, scale, and filter operations, empty gettext domains, invalid
Intl locales or `ResourceBundle` offsets, and malformed mbstring maps or
encodings. Validate arguments or catch the exception instead of expecting
warnings or permissive coercion.

### mbstring indices on malformed text (8.4-migration)

For strings with encoding errors, `mb_substr()` interprets character indices
consistently with other mbstring functions, so offsets from `mb_strpos()` can
be reused. SJIS-Mac indices refer to Unicode code points produced by
conversion, including characters that expand to multiple code points.

## PCRE syntax and compatibility

### PCRE 10.44 compatibility (8.4-migration)

PCRE2 10.44 recognizes `{,3}` as a quantifier rather than literal text. Some
character classes also changed meaning in UCP mode. Audit patterns that relied
on either behavior.

### PCRE 10.44 syntax (8.4.0)

PCRE2 supports variable-length lookbehind, permits spaces between braces in
Perl-compatible items, and raises the named-capture limit from 32 to 128
characters. The `r`/`(?r)` modifier, combined with `i`, prevents ASCII and
non-ASCII characters from mixing in caseless matches.

### PCRE and internationalization behavior (8.5-migration)

PCRE is built without `PCRE2_EXTRA_ALLOW_LOOKAROUND_BSK`; revise patterns that
use that extension. Intl regular collation sort handles numeric strings like
the standard `SORT_REGULAR`, and mbstring uses Unicode 17.0 data.

## Locale and formatting

### Formatting and locale behavior (8.5-migration)

A `printf`-family formatter without explicit precision treats precision as
zero instead of resetting it. Passing integer `0` as the locales argument to
`setlocale()` is unsupported and throws `TypeError`.

### Locale-aware list formatting (8.5.0)

`IntlListFormatter`, available with ICU 67 or newer, formats localized AND,
OR, or unit lists in wide, short, or narrow forms using its `TYPE_*` and
`WIDTH_*` constants.

## Image formats and dimensions

### Image formats and dimension units (8.5.0)

EXIF supports `OffsetTime*` tags and HEIF/HEIC. `getimagesize()` recognizes
HEIF/HEIC and, with ext-libxml, SVG. Image-size results include `width_unit`
and `height_unit`; they default to pixels but may differ. SVG is also
recognized by `image_type_to_extension()` and `image_type_to_mime_type()`.

