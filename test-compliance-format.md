# About conformance testing

This introduces a simple format to capture the expectations on any implementation of these template extensions.

Its purpose is to be 
* not too complicated too write / read the actual rules expressed
* not too complicated to have them automatically validated and tested


## Parts of the test-compliance format

These independant conformance testing files are composed of multiple parts that all fit four types.

Each of these start with a dedicated start-line that is recognisable by their leading character:
  * `#` for comments
  * `=` for assignments
  * `?` for templates
  * `$` for results

The lines following these starting lines make up the actual 'content' of the section. 

This means that the start of a new section implies that the content of the previous one is complete. Stated in reverse: one completes the content of any section by simply starting a new one, e.g. by a single `#`-sign on a line

Any remaining characters on the section starting-line itself is not considered part of the content (and can be ignored).

Depending on the type of section, the content is to be applied in different ways:

### The content of `#`-comment-section

This content has no effect and is to be ignored. Note the starting-line of such section will at lease finalise and close up the previous one.

As such, during sequential processing of the conformance test file it does not redefine (or reset) either the assigned context or the accumulated template-results.

### The content of `=`-assign-section

The content of the assign section is to be parsed as a json-dict that describes the context-variables to be made available to the template-evaluation.

The specified variables are to be extending / overwriting a pre-agreed `BASE_CONTEXT` dictionary that also contains implementation native datatypes used in these tests.

In case the content is empty, it will be considered the equivalent of `{}` and thus will lead to a simple clone of th `BASE_CONTEXT` dict to be applied.

In case no assign section is present, the same `BASE_CONTEXT` will be available during the processing of all template-sections.

Any new assign-section will lead to a reset / reconstruction of the context-variables to be used. This however does not have an effect on accumulated template-results still to be checked.

### The content of `?`-template-section

The content of the template-section is to be seen as a template that can be expanded while applying the resulting context set by any previous assign-section.

Errors during this evaluation should lead to a failure and break off further processing of the compliance test.

The expanded result of this evaluation is to be accumulated in a growing list of template-results that all can be checked simultaneously.

The evaluation will not bear any side-effects on the available context-variables.

### The content of `$`-result-section

The content of the result-section literally captures the expected output of any template-section that precedes it.

The prime effect of "executing" this section is that any earlier accumulated template-results up to that moment are compared to its content, thus actually implementing the conformance-validation.  Invalid results should lead to a test-failure and an extensive logging of which template filename and linenumber.

When all results do match, the list of accumulated results is reset, and further conformance testing can proceed.

Any defined context-variables (by assign-section) will not be changed.

### Regular usage pattern for tests

As the format is to be evaluated from top to bottom, the typical sequence will be to have multiple blocks that all folow this pattern:

* an opening `#`-comment-section
* an initialising `=`-assignment-section
* a number of variant `?`-template-sections that all are expected to produce the same result
* a single `$`-result-section that captures the expected results for the previous templates
* at least one, typically empty `#`-comment-section to wrap the result nicely 

Example:
```
= -- this assignment adds variable 'x' to the context
  {
    "x": "value of x"
  }
? -- this template will be expanding the variable
  using x = >{{ x }}< 
? -- this template will be expanding from the inline literal itself
  using x = >{{ 'value of x' }}< 
? -- this template is not doing any expansion at all
  using x = >value of x<
$ -- all the above should produce
  using x = >value of x<
# -- just forcefully ending the above $-results-section
```


## Aggreed contents of the BASE_CONTEXT 

During template evaluation there is so called execution 'context' present.
This in memory structure (typically a dictionary or hashmap) simply represents the available series of variables that can be referred to in the templates' `{{ expansion-statements }}`

The assignment-section purpose is to augment on this execution 'context'-dictionary.
The chosen textual syntax to do this is json.
This has great support for parsing across many programming-languages, but offers very little guarantee towards creating specific language-specific native datatypes.

To overcome this limitation the evaluation of these conformance tests assumes the availability of a default `BASE_CONTEXT` that introdces a guaranteed set of native typed variables in the execution context.

```python

BASE_CONTEXT: dict = {
    "txt_1": "1",
    "txt_001": "001",
    "txt_1_dot_0": "1.0",
    "txt_esc": "TestIng \\\"'escaper",
    "txt_none": "",
    "txt_space": " ",
    "txt_dquote": "\"",
    "txt_squote": "'",
    "txt_bslash": "\\",
    "txt_tab": "\t",
    "txt_nl": "\n",
    "txt_long": """This is a long text
that spans multiple lines
and contains 'single' and "double" quotes
and a backslash \\.""",
    "txt_false": "false",
    "txt_true": "true",
    "txt_any": "anything",
    "txt_May6_1970": "1970-05-06",
    "txt_Sep25_2025_5pm": "2025-09-25T17:00:00",            # time, anywhere
    "txt_Sep25_2025_5pm_loc": "2025-09-25T17:00:00+02:00",  # time in UTC+2

    "bool_t": True,
    "bool_f": False,

    "int_0": 0,
    "int_1": 1,
    "int_11": 11,
    "int_m111111": -111111,

    "float_0": 0.0,
    "float_1": 1.0,
    "float_m1": -1.0,
    "float_1_5": 1.5,
    "float_pi": math.pi,

    "dt_May6_1970": datetime.date(1970, 5, 6),
    "dt_Sep25_2025": datetime.date(2025, 9, 25),
    "dttm_Sep25_2025_5pm": datetime.datetime(2025, 9, 25, 17, 0, 0),
    "dttm_Sep25_2025_5pm_loc": datetime.datetime(
        2025, 9, 25, 17, 0, 0,
        tzinfo=datetime.timezone(datetime.timedelta(hours=2))
    ),

    "none": None,  # null, undefined, ...

    "list_none": [],

    "dict_none": {},
    "dict_map": [
        {"char": c, "num": n}
        for c, n in zip(string.ascii_lowercase, range(1, 27))
    ],
    "dict_mapables": [
        {"id": "ais1",  "col_char": "a"},
        {"id": "nis14", "col_char": "n"},
        {"id": "yis25", "col_char": "y"}
    ],
    "dict_john": {"name": "Doe", "given": "John", "age": 52, "alive": True, "score": 1.5, "born": datetime.date(1970, 5, 6)},
    "dict_jane": {"name": "Roe", "given": "Jane", "score": 1.7, "born": datetime.date(1975, 8, 15)},
}

```

## Formal format description

```ebnf

compliance-content := compliance-lines*

compliance-line := ignorable-whitespace? (empty-line | section-start-line | content-line) ignorable-whitespace? NEWLINE

NEWLINE := '\n'

# content lines are compared after trimming leading/trailing whitespace
ignorable-whitespace := (' ' | '\t')+

empty-line := ''

section-start-line := comment-start-line | assignment-start-line | template-start-line | result-start-line

comment-start-line := '#' remainder
assignment-start-line := '=' remainder
template-start-line := '?' remainder
result-start-line := '$' remainder

remainder := .* #this remainder works as the label / title of the section / but is ignored, not part of the 'content'

content-line := [^#=?$] .*
```

- content-lines are implied as being part of, completing, filling whatever section that got started before it
- this means any orphaned content-lines at the top of the file, preceeding the first section-start-line will be simply ignored / discarded


## Known limitations

### Result-lines can not (easily) start with any of the `#=?$` chars
This format does not allow to have content-lines that start with one of the control chars #=?$. 

However, by indenting content-lines (two chars recommended for readability) this can simply be overcome. 
This shifts these reserved characters away from the first column, making them no longer play their section-starter-role.
At these position, they can simply remain and effectively be part of valid content of any section, and thus also expected to be compared as the outcome of a test.

### Result-lines can not be empty (or only white-chars)

This format trims leading and trailing whitechars and ignores empty content-lines.  

As a consequence any (non-zero) number of them in a template-result will never match the captured expected result in a `$`-result-section

We stick to this limitation because the invisble nature of whitespaces makes them more likely to have unintended side-effects. The main reasoning for this is that checking for whitespace effects is not the goal of this format. Additionally, it should be clear that mild levels of such tests are still possible by enclosing them within clearly visible marker characters. 

## Best practices for writing conformance test files

It is recommended to:

* Start independent test-blocks with a `=`-assignment-section to at least force a reset of the context-variables
* Indent content 2 chars as lines get trimmed anyway (i.e. leading and trailing whitespace is ignored)
* Allow control chars to be followed with additional text - it gets ignored, is not part of the content, and can help as clarifying the intent of the test.
* Provide multiple `?`-template-sections so they can all be checked versus one expected `$`-result-section following them.
* Testing for white-space effects is best surrounded with visible characters for clarity.

## Implementation suggestions

Some suggestions to those seeking to implement an automated test platform for the evaluation of these conformity tests (i.e. processing this format):

* Check the first character of lines to detect section-starts
* Trim content-lines to ignore leading and trailing whitespace
* Simply ignore empty lines (do not add them to content)
* As a counterpart, also remove empty lines produced in template results before comparing them to expected result-output.
* Note that comment-lines at least complete any previous section, so they require more attention than just ignoring them.
* Keep track of original file locations and line-numbers to provide helpful error-reports about what portions are not "conforming"

## Recommended usage scenario

TODO add text as suggested in [#3](https://github.com/vliz-be-opsci/subyt-template-extension-specification/issues/3)

To check your own implementations for compliance with these conformity-tests we recommend linking them into your project through git submodules.
This allows to stay in sync with the latest version of these tests.

``` bash
# get into your implmentation project and make sure you are synced up
$ cd to/your/implementation/repo
$ git pull

# set up the submodule
$ touch .gitmodules
$ git clone git@github.com:vliz-be-opsci/subyt-template-extension-specification.git ./to/relative/path/for/content
$ git submodule add git@github.com:vliz-be-opsci/subyt-template-extension-specification.git ./to/relative/path/for/content
$ git submodule init
$ git submodule update

# commit and push this setup to your project repo
$ git add .gitmodules ./to/relative/path/for/content
$ git commit -m "setting up gitmodule for subyt performance checks"
$ git push
```
An example of this can be found in the [py-sema project](https://github.com/vliz-be-opsci/py-sema) 