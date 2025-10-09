# subyt :: turtle-support-extensions for jinja3 templating (and its php port 'twig')

## About Semantic Uplifting by Templating  (su-by-t)

Many mechanisms exist to perform semantic-uplifting, i.e. the common depiction of any automated means for the production of RDF triples.

Generally these techniques are distributed across a continuum, in a solution space with many dimensions and extreme boundaries.
This means they are:
* custom made to handle one case, versus generic an reusable;
* aimed for a developer audience, versus more intended to serve domain or object-model experts;
* tied to a specific programming language, or open to independent implementations; and
* requiring different levels of "RDF" awareness.

In this space su-by-t targets to take a very pragmatical position:
* returning to proven web-template approaches for producing text/html and text/xml formats;
* adding simple extensions to support text/turtle;
* opportunistically fall back to existing cross-language implementations;
* allowing developers with a basis web-templating approach to integrate this in automated systems; and
* separating the actual template-writing to be done by domain experts with basis understanding of RDF, specifically in text/turtle.

This document describes the set of filters and functions these templating engines should implement to support text/turtle generation.

## Known Implementations

* [py-sema](https://github.com/vliz-be-opsci/py-sema) provides a python implementation that adds these extensions to jinja2

## About conformance testing

In order for these various template-engines to provide and validate conformant implementations of these subyt-template extensions a formal language-neutral test-compliance-format has been designed.

Our tests are of course following this format. They can be found in the various `*.test` files in this package.

The format itself is described in a separate [test conformance format](./test-compliance-format.md) document.

## Reference guide for subyt filters and functions

### `| xsd(typename[, quote="'"][,fb=None])` formatting filter

This filter produces n3-compliant syntax for literal values to have an explicit datatype-indicator.
In the process it checks the compatibility of the input, and ensures adequate formatting before output.

Depending on the chosen typename other "input-compatibility" rules apply

Arguments:
* `typename` Should match one of the described types in the following sub-sections. It is case-insensitive and can ommit the explicit 'xsd:' prefix for brevity. For some of these additional variants to specify the type are accepted as well.
* `quote` Indicates the single `'` or double `"` quote character to be used to capture the value. When ommitted the single-quote `'` is assumed as default.
* `fb` Allows to make the operation of the formatting 'lenient' to errors -- when unavailable (i.e. None/undefined/null) any processing error should make the templating as a whole break, when available it should be a string that represents the fall-back resulting output in case of those errors.

#### typename = `xsd:boolean`

Applies `^^xsd:boolean` formatting to the provided input.

Leading to either `'true'^^xsd:boolean` or `'false'^^xsd:boolean`

This requires the input to be one of the following:
* a native boolean
* a native string, in which case all of `''`, `'0'`, `'off'`, `'false'`, `'no'` (ignoring case) are considered `False` and all other non-empty strings resolve to `True`
* a native numeric value (integer or floating-point), in which case `0` and `0.0` are considered `False` and all other values are `True`

Any other input case, like:

* None, undefined, null, or template-implementation equivalents
* dates, datetime values,
* lists,
* dictionaries,
* other object instances

is incompatible and will lead to an error (or produce the `fb` value in lenient mode)

#### typename = `xsd:integer`

Applies `^^xsd:integer` formatting to the provided input.

Leading to output of the form `'-?[1-9][0-9]*'^^xsd:integer`

This requires the input to be one of the following:
* a native integer numeric value
* a native string for which the interpretation as integer works out and its round-trip to string format exactly matches the input.  This excludes surrounding whitespace or leading zeroes. The template should explicitely deal with those cases.

Any other input case, like:

* boolean values,
* numeric floating-point values,
* strings that do not follow the round-trip rule,
* None, undefined, null, or template-implementation equivalents,
* dates, datetime values,
* lists,
* dictionaries,
* other object instances

is incompatible and will lead to an error (or produce the `fb` value in lenient mode)

#### typename = `xsd:float` and typename = `xsd:double`

Applies `^^xsd:float` respectively `xsd:double` formatting to the provided input.

Leading to output of the form `'-?[0-9]*\.[0-9]*'^^xsd:(float|double)`

The processing of both follows the same rules. `double` should probably be the principle choice, but since the distinct `xsd:float` type does exist it should be supported.

This requires the input to be one of the following:
* a native numeric floating-point value
* a native string, for which the interpretation as a floating-point value works out.

Any other input case, like:

* boolean values,
* numeric integer values,
* strings that do not parse as floating point values,
* None, undefined, null, or template-implementation equivalents,
* dates, datetime values,
* lists,
* dictionaries,
* other object instances

is incompatible and will lead to an error (or produce the `fb` value in lenient mode)

#### typename = `xsd:date`

Applies `^^xsd:date` formatting to the provided input.

Leading to output of the form `'YYYY-MM-DD'^^xsd:date`

This requires the input to be one of the following:
* a native date
* a native string, that strictly can be parsed as a iso8601 date "YYYY-MM-DD"

Any other input case, like:

* boolean, and numeric values,
* strings that are not strictly in the desired format,
* datetime values,
* None, undefined, null, or template-implementation equivalents,
* lists,
* dictionaries,
* other object instances

is incompatible and will lead to an error (or produce the `fb` value in lenient mode)

#### typename = `xsd:dateTime`

Applies `^^xsd:dateTime` formatting to the provided input.

Leading to output of the form `'YYYY-MM-DDThh:mm:ss[+tz:mm]'^^xsd:dateTime`

This requires the input to be one of the following:
* a native datetime value (which may or may not carry timezone information)
* a native string, that can be correctly parsed as a iso8601 datetime value (again with optional timezone part)

Any other input case, like:

* boolean, and numeric values,
* strings that are not strictly in the desired format,
* date values,
* None, undefined, null, or template-implementation equivalents,
* lists,
* dictionaries,
* other object instances

is incompatible and will lead to an error (or produce the `fb` value in lenient mode)

Note: The absence of a timezone fragment is considered an acceptable variant. As such it indicates a datetime value that is not actually localised but to happen 'in local time': i.e. either the context provides a geographic specific location elsewhere, or this value is considered to depict a recurring exact moment that appears across the globe as it occurs in each consecutive timezone.  This also means the formatting process should not inadvertantly impose its own locallity on the interpretation, and thus not add a timezone fragment if none is present in the output.

#### typename = `xsd:gYear`

Applies `^^xsd:gYear` formatting to the provided input.

Note: additional type-names include (case-ignoring) `yyyy` and `year`

Leading to output of the form `'-?0YYY'^^xsd:gYear`

See https://www.w3.org/TR/xmlschema11-2/#gYear

This requires the input to be one of the following:
* a numeric integer value,
* a native string, matching above integer numeric value
* a date value,

Any other input case, like:

* boolean,
* numeric floating-point values,
* strings that are not matching an integer value,
* datetime values,
* None, undefined, null, or template-implementation equivalents,
* lists,
* dictionaries,
* other object instances

is incompatible and will lead to an error (or produce the `fb` value in lenient mode)

#### typename = `xsd:gYearMonth`

Applies `^^xsd:gYearMonth` formatting to the provided input.

Note: additional type-names include (case-ignoring) `yyyy-mm` and `year-month`

Leading to output of the form `'-?0YYY-MM'^^xsd:gYearMonth`

See https://www.w3.org/TR/xmlschema11-2/#gYearMonth

This requires the input to be one of the following:
* a native string, that matches the actual yyyy-mm format
* a date value

Any other input case, like:

* boolean, and numeric values,
* strings that are not matching the pattern
* datetime values,
* None, undefined, null, or template-implementationn equivalents,
* lists,
* dictionaries,
* other object instances

is incompatible and will lead to an error (or produce the `fb` value in lenient mode)

#### typename = `xsd:anyURI`

Applies `^^xsd:anyURI` formatting to the provided input.

Leading to output of the form `'uri-conform-string'^^xsd:anyURI`

This requires the input to be one of the following:
* a native string that is conforming to a valid uri format, if needed by inserting some %HEX-encoding parts

The applied %HEX-encoding will be part of the output.

Any other input case, like:

* non-conforming uri strings,
* boolean, numeric, date, datetime values,
* None, undefined, null, or template-implementation equivalents,
* lists,
* dictionaries,
* other object instances

is incompatible and will lead to an error (or produce the `fb` value in lenient mode)

#### typename = `xsd:string`

Applies `^^xsd:string` formatting to the provided input.

Leading to output of the form `'value-text'^^xsd:string`

This requires the input to be one of the following:
* any native scalar format with straightforward textual representation,

Any other input case, like:

* None, undefined, null, or template-implementation equivalents,
* lists,
* dictionaries.

is incompatible and will lead to an error (or produce the `fb` value in lenient mode)

Note that 'other object instances' are accepted in this case and will be simply 'stringified' according to whatever their coded native string representation is.

Note about escaping: In turtle both `'` (single) and `"` (double) quotes are allowed to surround strings. This leads to those characters themselves to require backslash-`\`-escaping to be accepted in between them.  Additionally this means a single `\` itself needs to be represented as `\\`.

Note about multiline-strings: Multiline string values (i.e. containing `\n` newlines) require a triplet-variant of the quote to be used. Making the string-start-end markers actually being `'''` or `"""`.

#### typename = `xsd:@language-code`

Applies `@language-code` formatting to the provided input.

Leading to output of the form `'my text'@en`, `'mijn tekst'@nl`, and `'mon texte'@fr`

For this case the same requirements known from the `xsd:string` case mentioned above for
* the restrictions to valid inputs, as well as
* the notes around expected valid output formatting.

#### typename = `auto-date`

Inspects the input and applies the best (most precise) possible date-type to format it.
So, one of the following will be added (in order of preference / most restrictive first):
* `^^xsd:dateTime`
* `^^xsd:date`
* `^^xsd:yearMonth`
* `^^xsd:year`

This requires the input to be one of the following:
* a dateTime or date object
* an int that can be used as year
* a string that can be parsed into one of the supported date formats, i.e. iso8601 with or without time and timezone fragments, or matching either YYYY-MM, or a integer to be used as year

Any other input case, like:

* Strings that do not fit an understandable date format
* None, undefined, null, or template-implementation equivalents,
* lists,
* dictionaries,
* other object instances

is incompatible and will lead to an error (or produce the `fb` value in lenient mode)

#### typename = `auto-number`

Inspects the input and applies the best (most precise) possible available type to format it.
So, one of the following will be added (in order of preference / most restrictive first):
* `^^xsd:integer`
* `^^xsd:double`

This requires the input to be one of the following:
* a native numeric format, be it integer or floating-point
* a native string that can be parsed in one of those

Any other input case, like:

* Strings that do not fit an understandable numeric format
* None, undefined, null, or template-implementation equivalents,
* lists,
* dictionaries,
* other object instances

is incompatible and will lead to an error (or produce the `fb` value in lenient mode)

#### typename = `auto-any`

Inspects the input and applies the best (most precise) possible available type to format it.
So, one of the following will be added (in order of preference / most restrictive first):
* `^^xsd:integer`
* `^^xsd:boolean`
* `^^xsd:double`
* `^^xsd:dateTime`
* `^^xsd:date`
* `^^xsd:yearMonth`
* `^^xsd:string`

Note: `^^xsd:year` will not be produced as any input that fits this type would have already be considered as an `^^xsd:integer` already.

This support most of the inputs you could imagine:
* a native boolean
* a native numeric value
* a native date or datetime value
* a string value
* any object type that has a string representation

Any other input case, like:

* None, undefined, null, or template-implementation equivalents,
* lists,
* dictionaries.

is incompatible and will lead to an error (or produce the `fb` value in lenient mode)

### `| uri` filter

The `| uri` filter works very similar to the `| xsd('anyURI')` in the sense that it will check an safeguard correct URI formatting (and proper %hex-encoding) to the input.

The main difference though is that this will not add the `"uri-here"^^xsd:anyURI` formatting, but instead ensuring the `<uri-here>` format of adding angle-brackets for references in turtle.


### `uritexpand(template[, context ])` function

The `uritexpand()` function allows to expand [RFC 6570 uritemplates](https://datatracker.ietf.org/doc/html/rfc6570) by applying available values from the context passed.

Arguments:
* `template` a string holding an RFC 6570 uritemplate with `{variable}` regions to be expanded
* `context` is an optional record (dictionary) that provides in its fields the values to expand the uritemplate

Note if the context record is not provided, the available global template-context is used.

A typical application of this function is to combine it with the `| uri` filter.


### `regexreplace(pattern, replace, content)` function

The `regexreplace()` function allows to match and replace parts of a string.

Arguments:
* `pattern` a string that holds the [regular-expression](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_expressions/Cheatsheet) used to search for matching parts that need to be replaced
* `replace` a string holding the literal content that is to replace the matched parts
* `content` a string holding the input to perform the match-and-replace upon


### `map(mapping_data, fromname, toname[, cachekey])` constructor and `map.apply(record, fromname, toname)` function

The `map()` function produces a reusable (and cacheable) object, dubbed a `ValueMapper`, that can in turn be used to map field-values in records.

Arguments:
* `mapping_data` a list of records (typically dictionaries) expected to provide the keys and values to be used in the mapping
* `fromname` string name of a field in the mapping-data records. The values of this field are to be used as lookup keys for the actual mapping
* `toname` string name of another field in the mapping-data records. The values of this field are to be applied as the resulting value in the actual mapping
* `cachekey` (optional) string identifier for the generated mapping object

The caching is there to allow reusing an earlier instance of the created ValueMapper during processing, without the need to rebuild it in full.
This a performance enhancing measure only, it should not effect the outcome: starting from the same mapping_data should always lead to same mapping-results.
It is entirely up to the implementation how it deals with maintaining this cache, and any possible invalidation rules it might apply.

The provided `ValueMapper` object is expected to have a single method `apply(record, origin_name, target_name[, fallback])`

Arguments:
* `record` an object (typically dictionary) that holds various fields, of which one holding a value that needs the mapping. This instance must support setting a new value for the mapping result too.
* `origin_name` a string name of the field inside the record that holds the value to be mapped
* `target_name` a string name of the field inside the record that will be updated (even created) to hold the mapping-result
* `fallback` (optional) the value to use as result into the target_name field case the value in the origin_name field is not known as a key inside the `ValueMapper`



### `unite(expression1, expression2[, expressionN]*[,n=3][,sep=' '][,fb=''])`

The `unite(...)` function is designed to "guard and guarantee" all parts of an expression are there before joining them in the rendered output.

In its basic form it takes string arguments and joins them together (using the `sep` characters, which defaults to " " \[space\]), but only does so if all are none-blank (i.e. non-empty after strip) and if their total number does not exceed `n` (defaults to 3). In case those tests fail the `fb` (fallback value, defaulting to `''`) is rendered in stead.

In its extended form, additional non-string arguments can be passed as well. These are ignored in the rendering of the output but are evaluated to boolean (True/False) values that allow for additional conditions: only in the case where all of them evaluate to `True` the joined output will be the result. One `False` leading to the `fb` being rendered in stead.

The intended use of this is to avoid extensive `{% if .. endif %}` statements in the templates to check for conditions in the data being used in the template.

Typical usage is testing for properties in order to avoid dangling-for-empty-parts in statements of either:

- «?p ?o;» completions of triples, when a predicate is added in anticipation of a possibly missing object.
```
   {{ unite( 'pfx:when', optional_value  | xsd('dateTime') )}};
```

- «prefix:remainder» combinations, when a prefix and colon are left dangling if no actual remainder is there.
```
   {{ unite( unite('pfx', optional_remainder, sep= ':'), 'value'@en) }};
```

Arguments:
* `expression1 .. expressionN` comprise the actual content to be joined (if strings) and checked (as booleans). Next to these additional named arguments can be passed.

Named Arguments:
* `n=3` the maximum number of strings that will be joined, defaults to 3
* `sep=' '` the separator to apply in the join, default to ' '
* `fb=''` the output to be generated in stead of the string-join (when one of the bools evaluates to false, or the number of strings exceeds `n`)
