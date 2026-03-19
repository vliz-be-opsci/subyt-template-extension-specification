# SU-by-T Template Authoring Guide

This page provides practical tips and best practices for writing SU-by-T templates.
Where the [specification](specification.md) describes **what** the available filters and functions provide, this guide focuses on **how** to use them effectively — and hints at **why** these practices matter.

The rules are numbered so they can be referenced directly (e.g. *"this template violates T03"*).
Each rule follows the same pattern:

> **code** · title · short description · code example(s) · consequences if the rule is not followed

---

## Template-Writing Rules

### T01 · Comment Prelude

**Add a formal comment block at the top of every template file.**

Every template file should open with a structured comment that describes:

- the template name and purpose,
- the primary input variable (the iterated record), and
- any additional sets/dependencies it relies on.

This makes the intent of the template self-documenting and allows tooling to extract metadata without executing the template.

```jinja
{#-
Template:    my-type.ttl
Description: Produces RDF triples describing MyType instances.
Input:       path/to/main-input.csv  (iterated as `_`)
Sets:
    - path/to/lookup.csv  as lookup
Subyt settings:
    iteration:   true   # true (default) = one template render per input row;
                        # false = render template once for the entire input set
    ignorecase:  true   # true (default) = all input keys lowercased automatically;
                        # false = keys used as-is (case-sensitive)
    flatten:     true   # true (default) = nested dict keys joined with '.' separator;
                        # false = nested dicts kept as-is
-#}
```

!!! failure "Consequences if you skip this"
    - Another developer (or yourself later) must read through the whole template to understand its inputs and purpose.
    - Automated documentation or dependency-graph tooling cannot extract metadata.
    - Template collections become hard to maintain as they grow.

---

### T02 · Use `.ttl` Extension

**Name your turtle-producing template files with a `.ttl` extension** (e.g. `my-type.ttl` or `my-type.ldt.ttl`).

This signals to the SU-by-T runtime that HTML/XML auto-escaping must be turned off.
Without this, angle-brackets (`<`, `>`) in URI references, caret (`^`) characters in typed literals, and other turtle syntax characters may be incorrectly escaped or mangled.

The double-extension variant `*.ldt.ttl` is a common convention to make the template-engine origin explicit at a glance (ldt = linked data templating), while still keeping the `.ttl` suffix that controls escaping behaviour.

```
templates/
  my-type.ttl          # ✓ HTML-escaping off, ready for turtle output
  my-type.ldt.ttl      # ✓ also acceptable – origin explicit
  my-type.j2           # ✗ HTML-escaping on by default – angle brackets will break
```

!!! failure "Consequences if you skip this"
    - URI references (`<http://example.org/thing>`) are HTML-escaped into `&lt;http://...&gt;`, producing invalid Turtle.
    - Typed literals (`'value'^^xsd:date`) may have their `^` characters escaped.

---

### T03 · Do Not Overwrite Built-in Filters or Functions

**Never define a macro or variable that shadows a built-in SU-by-T filter or function.**

The names `xsd`, `uri`, `uritexpand`, `regexreplace`, `map`, and `unite` are reserved by the SU-by-T extension layer.
Redefining them in your template will silently replace the built-in behaviour with your custom logic, causing subtle bugs that are hard to trace — especially for future maintainers who rely on standard behaviour.

```jinja
{# ✗ BAD — overwrites the built-in `uri` filter #}
{% macro uri(value) %}{{ value | replace(' ', '%20') }}{% endmacro %}

{# ✓ GOOD — use a distinguishing name for custom helpers #}
{% macro safe_local_id(value) %}{{ value | replace(' ', '%20') }}{% endmacro %}
```

!!! failure "Consequences if you skip this"
    - Built-in validation (e.g. `| uri` rejecting malformed URIs) is silently bypassed.
    - Templates that import or include your file also inherit the broken override.
    - Test failures caused by the override will be very hard to diagnose.

---

### T04 · Centralise Identifier Logic in a Shared Macro File

**Provide a dedicated include file that defines macros for constructing URIs for each entity type in a consistent manner.**

URI construction logic often needs to be reused across multiple templates in the same project.
Centralising it in one file ensures all templates produce the same identifier for the same entity and makes it easy to update the pattern in one place.

Two folder conventions are commonly used — choose one and apply it consistently:

- **`includes/`** — groups all reusable snippets (prefixes, macros) together, close to standard Jinja usage (e.g. `includes/identifiers.ttl`, `includes/prefixes.ttl`).
- **`macros/`** — separates macro files from other includes, making the distinction between "prefix declarations" and "callable macros" explicit (e.g. `macros/identifiers.ttl`, `prefixes/prefixes.ttl`).

```jinja
{# includes/identifiers.ttl #}
{% macro station_uri(station_id) -%}
  {{ uritexpand("http://example.org/station/{id}", {"id": station_id}) | uri }}
{%- endmacro %}
```

```jinja
{# my-template.ttl — import and use the shared macro #}
{%- from './includes/identifiers.ttl' import station_uri %}

{{ station_uri(_.station_id) }}
    a ex:Station ;
    rdfs:label {{ _.name | xsd('string') }} ;
.
```

!!! failure "Consequences if you skip this"
    - URI patterns for the same entity type drift between templates over time.
    - A single change to a URI pattern requires edits in many files.
    - Inconsistent identifiers break `owl:sameAs` chains and federated queries.

---

### T05 · Apply `| xsd` to All Literal Values

**Wrap every object-literal with `| xsd(typename)` to ensure correct type formatting and runtime checking.**

The `| xsd` filter validates the input value against the declared XSD type, formats it properly for turtle, and makes the datatype annotation explicit in the output.
Omitting it produces untyped plain literals, which are both harder to query and may silently drop precision (e.g. an integer rendered as a bare string).

```jinja
{# ✗ BAD — plain untyped literals, no validation #}
ex:thing ex:count {{ _.count }} ;
         ex:label {{ _.label }} ;
         ex:date  {{ _.date }} .

{# ✓ GOOD — typed and validated #}
ex:thing ex:count {{ _.count | xsd('integer') }} ;
         ex:label {{ _.label | xsd('string') }} ;
         ex:date  {{ _.date  | xsd('date') }} .
```

Use `fb=''` (fallback) to gracefully skip optional properties that may be absent:

```jinja
ex:thing ex:optionalScore {{ _.score | xsd('double', fb='') }} .
```

!!! failure "Consequences if you skip this"
    - Numeric values are serialised as plain strings (`"42"` instead of `"42"^^xsd:integer`).
    - Invalid values (e.g. a non-date string passed to a date field) will silently produce malformed output instead of raising an error at generation time.
    - SPARQL queries relying on typed comparisons (`FILTER(?count > 10)`) will not work correctly.

---

### T06 · Apply `| uri` to All URI References

**Wrap every URI reference with `| uri` to guarantee well-formed angle-bracket notation and correct percent-encoding.**

The `| uri` filter validates the input as a proper URI, applies any needed percent-encoding, and wraps it in `<…>` for turtle.
Without it, spaces, non-ASCII characters, or other special characters in a URI value will produce invalid Turtle.
Chain `| uri` after `uritexpand()` for the cleanest pattern (see T07).

```jinja
{# ✗ BAD — raw string, no validation, no angle brackets #}
ex:thing owl:sameAs {{ _.external_uri }} .

{# ✓ GOOD — validated, encoded, and angle-bracketed #}
ex:thing owl:sameAs {{ _.external_uri | uri }} .
```

!!! failure "Consequences if you skip this"
    - A URI containing spaces or non-ASCII characters produces invalid Turtle.
    - Parsers / linters will reject the output.
    - Missing `<…>` delimiters cause the URI to be interpreted as a prefixed name.

---

### T07 · Use `uritexpand()` to Construct URIs — Never String Concatenation

**Build URIs from templates using `uritexpand()` instead of string concatenation.**

`uritexpand()` expands [RFC 6570 URI templates](https://datatracker.ietf.org/doc/html/rfc6570), properly percent-encoding each variable value before inserting it.
Plain string concatenation does not encode the parts, so a value containing `/`, `?`, `#`, or spaces will silently produce a syntactically or semantically broken URI.

```jinja
{# ✗ BAD — concatenation does not percent-encode values #}
ex:thing ex:hasPage <http://example.org/pages/{{ _.title }}> .

{# ✓ GOOD — uritexpand handles encoding, | uri wraps in angle brackets #}
ex:thing ex:hasPage {{ uritexpand("http://example.org/pages/{title}", _) | uri }} .
```

!!! failure "Consequences if you skip this"
    - Special characters in field values silently break the URI.
    - The resulting URIs may differ across implementations that handle template expansion differently.
    - Federated queries and owl:sameAs links fail when URIs are not canonical.

---

### T08 · Use `unite()` to Replace Conditional Blocks

**Prefer `unite()` over `{% if … %}…{% endif %}` blocks when guarding optional property-value pairs.**

`unite()` tests that all its string arguments are non-blank before joining them.
This eliminates the visual noise of repeated if/endif blocks around optional predicates and avoids dangling semicolons or commas when a value is absent.

```jinja
{# ✗ NOISY — manual if/endif for every optional property #}
{%- if _.start_date and _.start_date != '' %}
ex:thing ex:startDate {{ _.start_date | xsd('date') }} ;
{%- endif %}

{# ✓ CLEAN — unite() handles the guard in one line #}
    {{ unite('ex:startDate', _.start_date | xsd('date', fb='')) }} ;
```

For a URI reference pair (prefix + local part) that should only appear together:

```jinja
    {{ unite( unite('pfx', optional_local_part, sep=':'), 'ex:predicate', sep=' ') }} ;
```

!!! failure "Consequences if you skip this"
    - Verbose if/endif blocks obscure the intent of the template.
    - A missing endif or wrong nesting creates whitespace or syntax errors in the output.
    - Dangling semicolons (`;`) at the end of a triple block cause invalid Turtle.

---

## Validation Tips

### V01 · Test with Both Typical and Edge-Case Data

**Run your template against both representative real-world data and deliberately challenging inputs.**

Good test data covers:

- **Happy path** — the normal, expected input values.
- **Missing / null values** — fields that are empty, `null`, `"NA"`, or absent from the record.
- **Boundary values** — very long strings, extreme numbers, dates at month/year boundaries.
- **Special characters** — quotes, backslashes, newlines, non-ASCII in string fields; slashes and hashes in URI fields.
- **Type mismatches** — a number where a date is expected, a boolean where a string is expected.

Deliberately try to break the template.
If it doesn't break cleanly (raising an informative error), that is itself a finding to address.

The `.test` files in this specification repository provide a useful reference for the pattern of pairing inputs with expected outputs — the same approach can be applied to your own templates: define an input record, run the template, and capture the expected Turtle output as a reference to check against.

!!! failure "Consequences if you skip this"
    - Edge-case inputs that were never tested silently produce malformed or empty output at production time.
    - A template that never raises errors on bad input may be hiding silent data loss.

---

### V02 · Validate the Generated Turtle / TriG Output

**Always parse the generated output with a dedicated Turtle or TriG parser/linter before deploying.**

The template engine cannot guarantee that the combined output of all triples forms syntactically valid Turtle.
An external parser catches issues such as missing prefixes, unclosed blank nodes, dangling semicolons, or encoding errors that are invisible in the raw template.

Recommended tools:

| Tool | Format | Notes |
|------|--------|-------|
| [riot](https://jena.apache.org/documentation/io/) (Apache Jena) | Turtle, TriG, N-Triples, … | `riot --validate my-output.ttl` |
| [rapper](http://librdf.org/raptor/rapper.html) (Raptor) | Turtle, N-Triples, … | `rapper -i turtle my-output.ttl` |
| [rdfpipe](https://rdflib.readthedocs.io/en/stable/intro_to_parsing.html) (rdflib) | Turtle, TriG, … | `rdfpipe -i turtle my-output.ttl` |
| [EasyRDF online validator](https://www.easyrdf.org/converter) | Turtle | browser-based |
| [TTL validator](http://ttl.summerofcode.be/) | Turtle | browser-based |

Integrate this step into your CI pipeline so every generated file is validated automatically.

!!! failure "Consequences if you skip this"
    - Syntactically invalid output reaches consumers who then face cryptic parser errors.
    - Silent encoding or quoting bugs are discovered only in production.
    - Downstream tooling (triple stores, reasoners) rejects or partially loads the file.
