# XML, SOAP, Text, and Media

Use this reference for XML-family extensions, SOAP serialization, Unicode and
locale behavior, images, and native callback registration. Included batch
identifiers cited here are `8.4-migration`, `8.4.0`, `8.5-migration`, and
`8.5.0`.

## XML callbacks and object association

In `8.4-migration`, XML handler setters enforce an effective
`callable|string|null` handler type. A legacy method-name string is resolved
only after `xml_set_object()` associates an object.

Both `xml_set_object()` and non-callable method-name strings passed to
`xml_set_*()` are deprecated. Migrate to direct callables:

```php
[$handler, 'method']
```

`xml_parser_free()` is deprecated in `8.5-migration`, because the parser object
is released automatically.

## DOM and XPath

In `8.4-migration`, `DOMXPath` cannot be cloned,
`DOMImplementation::getFeature()` has been removed, `DOM_PHP_ERR` is
deprecated, and obsolete DOM encoding and configuration properties are
deprecated.

Since `8.4.0`, `DOMXPath::registerPhpFunctions()` accepts any callable.
`DOMXPath::registerPhpFunctionNs()` registers a callback under a namespace so
XPath can call it with native function syntax instead of
`php:function('name')`.

## XSL callbacks, limits, and parameters

Since `8.4.0`:

- `XSLTProcessor::registerPhpFunctions()` accepts any callable.
- XSLT parameters can contain both single and double quotes.
- `XSLTProcessor::$maxTemplateDepth` and
  `XSLTProcessor::$maxTemplateVars` constrain recursion depth and variable
  counts.

Since `8.5.0`, the `namespace` argument to
`XSLTProcessor::getParameter()`, `setParameter()`, and `removeParameter()`
takes effect. It supplies the namespace for an unqualified `name`; Clark
notation or a QName supplies the namespace through its URI or prefix instead.

## XML-family error behavior

In `8.4-migration`, XMLReader, XMLWriter, and XSL operations throw where
applicable for:

- invalid encodings;
- embedded NUL bytes;
- incompatible objects; and
- failed PHP callbacks.

Validate these inputs and catch exceptions at the document-processing boundary.

`XMLReader` extension constants also gained declared types in
`8.4-migration`.

## SimpleXML iteration and XPath

In `8.4-migration`, calling methods such as `asXML()` or `getName()`, or
casting a `SimpleXMLElement` to string, no longer implicitly resets its
iterator. Code that depended on this side effect must call `rewind()`.

In `8.5-migration`, `SimpleXMLElement::xpath()` warns and returns `false` if
the expression evaluates to something other than a node set.

## SOAP objects, functions, and builds

### Internal member types

In `8.4-migration`:

- `SoapClient::$httpurl` is a `Soap\Url` object.
- `SoapClient::$sdl` is a `Soap\Sdl` object.
- `SoapClient::$typemap` is an array.

Replace resource checks on these members with null checks where absence is the
relevant state.

Passing `SOAP_FUNCTIONS_ALL` or another integer to
`SoapServer::addFunction()` is deprecated. Pass an array of function names,
such as a flattened `get_defined_functions()` result.

### Build dependency

SOAP optionally depends on the session extension in `8.4-migration`. A build
without session but with `--enable-rtld-now` can fail at startup when SOAP
loads. Avoid that flag combination or load the session extension.

### Class maps and date values

Since `8.4.0`, SOAP class-map keys can use Clark notation to distinguish
same-named types in different namespaces:

```php
$classMap = ['{http://example.com}foo' => 'FooClass'];
```

`DateTimeInterface` values supplied for `xsd:datetime` and related SOAP
elements serialize as date/time values rather than empty strings.

### URI parsing, schemas, and reason languages

In `8.5-migration`, `SoapClient::__doRequest()` has an optional URI-parser
class argument. `null` keeps `parse_url()` behavior; `Uri\Rfc3986\Uri` and
`Uri\WhatWg\Url` select the newer parser backends.

Since `8.5.0`, `SoapClient::__getTypes()` includes enumeration cases. SOAP 1.2
Reason Text supports `xml:lang`, exposed through an optional `lang` parameter
on `SoapFault::__construct()` and `SoapServer::fault()`.

## mbstring indices and Unicode data

For strings with encoding errors, `mb_substr()` interprets character indices
consistently with other mbstring functions in `8.4-migration`, allowing
offsets from `mb_strpos()` to be reused.

SJIS-Mac indices refer to Unicode code points produced by conversion, including
characters that expand to multiple code points.

In `8.5-migration`, mbstring uses Unicode 17.0 data. Test classifications,
case mappings, segmentation, and validation that depend on a Unicode version.

Malformed mbstring maps and encodings now raise `ValueError` in
`8.4-migration`, rather than relying on permissive fallback behavior.

## Internationalization

### Validation and configuration

In `8.4-migration`, invalid Intl locales and invalid `ResourceBundle` offsets
raise `ValueError`.

In `8.5-migration`:

- `intl.error_level` is deprecated; perform explicit error checks or enable
  `intl.use_exceptions`.
- Intl timezone and locale operations reject documented invalid states with
  exceptions.
- Regular collation sort handles numeric strings like standard
  `SORT_REGULAR`.
- Intl requires ICU 57.1 or newer.

### Localized lists

Since `8.5.0`, `IntlListFormatter` is available with ICU 67 or newer. It
formats localized AND, OR, or unit lists in wide, short, or narrow form using
the `TYPE_*` and `WIDTH_*` constants.

## Image metadata and sizing

In `8.4-migration`, invalid GD quality, speed, scale, and filter ranges raise
`ValueError`. The `SUNFUNCS_RET_*` constants are deprecated.

Since `8.5.0`:

- EXIF supports `OffsetTime*` tags and HEIF/HEIC.
- `getimagesize()` recognizes HEIF/HEIC and, when ext-libxml is available,
  SVG.
- Image-size results include `width_unit` and `height_unit`; they default to
  pixels but can differ.
- `image_type_to_extension()` and `image_type_to_mime_type()` recognize SVG.

`imagedestroy()` is deprecated in `8.5-migration`, because the image object is
released automatically.
