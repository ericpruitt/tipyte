Tipyte: Tiny Python Templating Engine
=====================================

Tipyte (pronounced "tippity") is a Python templating engine that supports
Python versions 2.7 and 3.0+. The syntax used for Tipyte is almost pure Python
with a small amount of syntactic sugar to keep the templates from being
whitespace sensitive and three template-specific built-in functions.

Tags
----

There are four types of templating tags in Tipyte: escaped expression tags,
unescaped expression tags, statement tags and comments. The following sample
template makes use of all four tags which are described in further detail
below:

    1   {# Author: Eric Pruitt #}
    2   {% is_day_time = 6 <= hour_of_day <= 18 %}
    3   <!doctype html>
    4   <html lang="en">
    5   <html>
    6       <head>
    6           <meta charset="utf-8">
    7           <title>{{ page_title }}</title>
    9           <link rel="stylesheet" type="text/css" href="default.css">
    10           {% if not is_day_time %}
    11              <link rel="stylesheet" type="text/css" href="night-theme.css">
    12          {% endif %}
    13          {% for name, content in meta_tags.items() %}
    14              <meta name="{{ name }}" content="{{ content }}">
    15          {% endfor %}
    16      </head>
    17      <body>
    18          <h1>Hello!</h1>
    19          <div id="content">
    20              Welcome to my website.
    21          </div>
    22          {= advertisement_footer =}
    23      </body>
    24  </html>

### Comment Tags ###

Comments are denoted by blocks of `{# ... #}`. The contents of comment tags are
ignored.

    1   {# Author: Eric Pruitt #}
    ...

### Expression Tags ###

Escaped expression tags are denoted by blocks of `{{ .... }}`. The result of
the expression within the tag will be rendered and then passed to the
template's escape function. The default escape method for all templates
produces HTML-safe output.

    ...
    7           <meta charset="utf-8">
    8           <title>{{ page_title }}</title>
    9           <link rel="stylesheet" type="text/css" href="default.css">
    ...
    13          {% for name, content in meta_tags.items() %}
    14              <meta name="{{ name }}" content="{{ content }}">
    15          {% endfor %}


Using blocks of `{= .... =}` will add the result of the expression to the
template verbatim without escaping.

    ...
    21          </div>
    22          {= advertisement_footer =}
    23      </body>
    24  </html>

The contents of expression tags _must_ be an expression; using statements
("import", "with", "while", etc.) will produce a syntax error.

### Statement Tags ###

Statement tags are denoted by blocks of `{% ... %}`. Anything that Python does
not evaluate as an expression must be enclosed in these tags. Unlike vanilla
Python, Tipyte is **not** whitespace sensitive. Statement blocks that would
normally require a new level of indentation are instead terminated with
"end..." blocks. For example, `{% with ... %}` must be paired with an
`{% endwith %}`, `{% if ... %}` paired with `{% endif %}`, etc. Trailing colons
(":") after "for", "while", "if", etc. are optional in Tipyte.

    ...
    2   {% is_day_time = 6 <= hour_of_day <= 18 %}
    ...
    9           <link rel="stylesheet" type="text/css" href="default.css">
    10          {% if not is_day_time %}
    11              <link rel="stylesheet" type="text/css" href="night-theme.css">
    12          {% endif %}
    13          {% for name, content in meta_tags.items() %}
    14              <meta name="{{ name }}" content="{{ content }}">
    15          {% endfor %}
    16      </head>
    ...

### Whitespace Truncation ###

All whitespace outside of tags is preserved by default, but this can be changed
by adding "-" immediately after any opening tag or immediately before any
closing tag. Assume the following block of code is the entirety of a template:

    <h1>   {{ "Hello" }} {{ "world!" }}    </h1>

The output would be:

    <h1>   Hello  world!    </h1>

By adding "-" in a couple of places, ...

    <h1>   {{- "Hello" }} {{ "world!" -}}    </h1>

...the output now becomes:

    <h1>Hello world!</h1>

Note that when adding "-" to tags, adjacent newlines will also be removed.

### Built-in Functions ###

The default scope of templates includes three built-in functions. They are
`include`, `raw_include` and `defined`.

#### include(path, raw=False, escaper=None) ####

Incorporate `path` into template output. If `raw` is `False`, the file will be
parsed as a template and executed, but if `raw` is `True`, the contents of the
file will be incorporated into the output verbatim. Normally, the escape
function of the calling template will be used to escape the included template,
but this can be overridden by setting `escaper`. If `path` is a relative path,
it will be interpreted as being relative to the directory of the calling
template. Note that this function does **not** return the included data.

#### raw_include(path) ####

Helper function to call `include` with `raw=True`; the following two
expressions are equivalent:

    {% raw_include("file.txt") %}
    {% include("file.txt", raw=True) %}

#### defined(name) ###

Return boolean value indicating whether or not a variable is defined. The
`name` is given as a string.

### What about ...? ###

- **... using a characters other than "{" and "}" for blocks?** This isn't
  supported out-of-the box right now, but only a handful of lines need to be
  modified to use a different symobl; search for `OPEN_TAGS` and `CLOSE_TAGS`
  in the source code.

- **... using literal strings that are the same as the block tags?** To insert
  a literal string that's the same one of the block tags, use `{= ... =}` with
  a string e.g. `{= "{=" =}`.

Python Interface
----------------

The two most important functions exposed by Tipyte are `template_to_function`
and `template_traceback`.

### template_to_function(path, escaper=html_escape) ###

Convert template into a callable function. By default, the template output will
be made HTML-safe, but the content escape method can be changed by setting the
`escaper` argument.

The resulting function action can be called using two different conventions to
pass state into the template. One way to pass state into the function is to
provide variable names and values as keyword arguments to the function:

    render_inbox = template_to_function("inbox.html")
    html = render_inbox(title="Inbox", email="j.doe@example.com")

Once execution is finished, any state defined only within the template is lost.
Alternatively, the template function can be called with a dictionary as an
argument:

    render_inbox = template_to_function("inbox.html")
    variables = {
        "title": "Inbox",
        "email": "j.doe@example.com",
    }
    html = render_inbox(variables)

Within the template, all members of the dictionary will be accessible as
variables. When template execution is finished, all of the template's state
will be preserved in the dictionary; any modified or newly defined variables
will be reflected in dictionary. Continuing from the example above, if the
template contained a statement like `{% name = ... %}`, the dictionary would
contain a new key after the template was rendered:

    >>> variables
    {'name': 'Jess Doe', 'email': 'j.doe@example.com', 'title': 'Home Page'}

To avoid conflicting with definitions used internally by the template system,
no user-defined variable names may start with `_template_`. If an error is
raised during template execution, the dictionary may contain internal variables
starting with this prefix.

### template_traceback(templates_only=False) ###

An exception raised inside of a template will produce a traceback that can be
hard to follow. When this function is called within an exception-handling
block, it returns a sanitized traceback in the form of a string that correctly
maps locations in the stack to the corresponding locations in the templates.

This is what a standard stack trace looks like when an exception is raised
within a template:

    Traceback (most recent call last):
      File "app.py", line 412, in <module>
        main()
      File "app.py", line 253, in main
        print(admin_page())
      File ".../tipyte:.py", line 376, in function
        exec(compiled_template, dict(), symbols)
      File "/._/python-templates/.../admin.html", line 3, in <module>
        <div class="container-fluid">
      File ".../tipyte.py", line 345, in include
        template_to_function(path, escaper=escaper)(symbols)
      File ".../tipyte.py", line 376, in function
        exec(compiled_template, dict(), symbols)
      File "/._/python-templates/.../user-list.html", line 3, in <module>
        <ul>
    NameError: name 'query' is not defined

This is what the sanitized stack trace returned by this function looks like:

    Traceback (most recent call last):
      File "app.py", line 412, in <module>
        main()
      File "app.py", line 253, in main
        print(admin_page())
      File ".../admin.html", line 5, in <module>
        {% include("user-list.html") %}
      File ".../user-list.html", line 4, in <module>
        {% for username, country, status in query("SELECT * FROM Users") %}
    NameError: name 'query' is not defined

Do not use this function when handling a `SyntaxError`. Any syntax errors
generated from within a template will already have been modified to include all
the information needed to easily determine where the syntax error is. When the
`templates_only` option is set, the only files that will be shown in the
traceback are templates; lines from pure-Python files will be elided. If
`templates_only` were set, the following traceback would be returned lieu of
the one above:

    Traceback (most recent call last):
      File ".../admin.html", line 5, in <module>
        {% include("user-list.html") %}
      File ".../user-list.html", line 4, in <module>
        {% for username, country, status in query("SELECT * FROM Users") %}
    NameError: name 'query' is not defined

Example usage:

    render_home_page = template_to_function("home.html")
    try:
        render_home_page(date="January 10th, 2016")
    except Exception as error:
        if not isinstance(error, SyntaxError):
            error.template_traceback = template_traceback()
        raise
