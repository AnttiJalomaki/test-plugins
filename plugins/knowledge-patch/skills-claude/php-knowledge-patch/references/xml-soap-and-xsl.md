# XML, SOAP, and XSL

Use this reference for DOM, XPath, XML handlers, SimpleXML, SOAP, and XSLT
behavior and migration.

## DOM and XML callbacks

### DOM and GMP object restrictions (8.4-migration)

`DOMXPath` cannot be cloned, and `DOMImplementation::getFeature()` has been
removed. `GMP` is final and cannot be subclassed.

### XML handler callables (8.4-migration)

XML handler setters enforce an effective `callable|string|null` handler type.
Legacy method-name strings resolve only after `xml_set_object()` associates an
object. Migrate to direct callables such as `[$handler, 'method']`;
`xml_set_object()` and non-callable strings are deprecated.

### Native callables in DOM XPath (8.4.0)

`DOMXPath::registerPhpFunctions()` accepts any callable.
`DOMXPath::registerPhpFunctionNs()` registers namespaced callbacks so XPath
can call them with native function syntax rather than
`php:function('name')`.

## SimpleXML behavior

### SimpleXML iteration no longer rewinds implicitly (8.4-migration)

Calling methods such as `asXML()` or `getName()`, or casting a
`SimpleXMLElement` to string, no longer resets its iterator. Call `rewind()`
explicitly when needed.

### SimpleXML and SOAP behavior (8.5-migration)

`SimpleXMLElement::xpath()` warns and returns `false` when the expression
produces something other than a node set. `SoapClient::__doRequest()` has an
optional URI-parser class argument: `null` keeps `parse_url()`, while
`Uri\Rfc3986\Uri` and `Uri\WhatWg\Url` select the new parser backends.

## SOAP types, serialization, and builds

### SOAP member type migrations (8.4-migration)

`SoapClient::$httpurl` and `$sdl` are `Soap\Url` and `Soap\Sdl` objects, while
`$typemap` is an array. Replace resource checks on these members with null
checks.

### SOAP build dependency (8.4-migration)

SOAP optionally depends on the session extension. A build without session but
with `--enable-rtld-now` can fail at startup when SOAP loads. Avoid that flag
combination or load the session extension.

### SOAP class maps with namespaces (8.4.0)

SOAP class-map keys may use Clark notation to disambiguate identically named
types from different namespaces.

```php
$classMap = ['{http://example.com}foo' => 'FooClass'];
```

### SOAP DateTime serialization (8.4.0)

`DateTimeInterface` instances supplied for `xsd:datetime` and similar SOAP
elements serialize as date/time values rather than empty strings.

### SOAP schema and reason-language support (8.5.0)

`SoapClient::__getTypes()` includes enumeration cases. SOAP 1.2 Reason Text
supports `xml:lang`, exposed through an optional `lang` parameter on
`SoapFault::__construct()` and `SoapServer::fault()`.

## XSLT parameters, callbacks, and limits

### Quote-safe XSL parameters (8.4.0)

XSLT parameters may contain both single and double quotes without failing.

### Native XSL PHP callbacks (8.4.0)

`XSLTProcessor::registerPhpFunctions()` accepts any callable.

### XSL evaluation limits (8.4.0)

`XSLTProcessor::$maxTemplateDepth` and
`XSLTProcessor::$maxTemplateVars` control recursion depth and variable limits
during XSL template evaluation.

### Namespace-aware XSL parameters (8.5.0)

The `namespace` argument of `XSLTProcessor::getParameter()`,
`setParameter()`, and `removeParameter()` is effective. It applies when `name`
is unqualified; Clark notation or a QName supplies the namespace through its
URI or prefix.

