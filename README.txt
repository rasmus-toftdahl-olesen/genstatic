genstatic - Generate and maintain static HTML files with Django templates

Genstatic is a tool for generating and managing sets of static HTML
(or other text format) files using the Django template system.  It
exists to save web designers time and sanity.

It's useful when you need to create a set of HTML files sharing
structural similarities, in a low-maintenance way.  You can do things
like change the doctype of 100 different HTML files, or change the
year of the copyright date in all their footers, by making a minor
change to one file.

It's also proven EXTREMELY useful when, as a web designer, you are
serving a client who hands you a very messy folder of static files for
their existing website.  You can use genstatic to bring order to that
chaos, and modify them with much less time and duplication of your
effort.

Genstatic is not limited to HTML files.  You can use it to generate
CSS, Javascript, plain text - essentially whatever textual format you
need.

Genstatic is based on Django templates, and lets you leverage some of
its powerful features: template inheritance, template variables, and a
rich built-in function library (Django's "tags" and "filters").  Learn
more at http://docs.djangoproject.com/en/1.2/topics/templates/ .

OVERVIEW

Usage:
  genstatic.py [options] srcdir destdir

For a full list of options, execute "genstatic.py -h".

srcdir and destdir are directories.  srcdir contains files - HTML,
CSS, Javascript, images, etc. Some or all of these files may be Django
templates.  srcdir normally also contains a magic subdirectory, named "_"
(underscore), that can contain inheritable base templates.  These are
all files you create.

Files in srcdir, other than those under "_", are rendered as Django
templates, and written into destdir (which is created, and by default,
not overwritten.  Override that with the --clobber option.)

Not all files will be rendered this way: only those with certain
filename extensions.  (By default, it's "htm", "html", "txt" and
"css".)  Files that don't match this are just copied straight through.
You can tell genstatic to match a different set of extensions with the
-x option.

Often, you will like to set variable names and values that are
available to the templates.  Create a python file that defines these
variables - for example, "myparams.py". Then pass this file name with
the -v option to genstatic.

(You must use Python syntax in this file.  Not a "Pythonista"? Don't
worry; if you're the type of person who wants to use genstatic, you
will probably find the variable syntax very easy to pick up.  See
Appendix A for a primer.)

Variable names starting with "gs_" are magic Genstatic variables with
special properties.  More on those below.

HOW ABOUT AN EXAMPLE

Suppose srcdir contains the files index.html, about.html, and
_/base.html, with the following contents:

# _/base.html:
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
  <head>
    <link rel="stylesheet" href="/widget.css">
    <script src="/widgets.js" type="text/javascript"></script>
    <title>{% block title %}{% endblock %}</title>
  </head>
  <body>
    <div id="content">
      {% block content %}
      <!-- page content goes here -->
      {% endblock %}
    </div>
    <div id="footer">Copyright {{copydate}} Joe's Widget Shop</div>
  </body>
</html>
# end of _/base.html

# index.html:
{% extends "_/base.html" %}
{% block title %}Welcome to Joe's Widget Shop{% endblock %}
{% block content %}
<h1>Joe's Widget Shop</h1>
<p>Welcome to Joe's Widget Shop, serving the greater Gotham, Idaho area!</p>
{% endblock %}
# end of index.html

# about.html:
{% extends "_/base.html" %}
{% block title %}About Joe's Widget Shop{% endblock %}
{% block content %}
<h1>About Joe's Widget Shop</h1>
<p>One warm summer night in '76, Joe had a vision: A clean and well-lit shop,
with widgets as far as the eye could see. It wasn't long before...</p>
{% endblock %}
# end of about.html

You also create a file named vars.py that just contains this line:
copydate=2010

Simply invoke on the command line:
  genstatic.py -v copydate.py srcdir destdir

destdir is created, containing index.html and about.html, and that's
it;  base.html is omitted.  Here's what those two files will contain:

# index.html:
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
  <head>
    <link rel="stylesheet" href="/widget.css">
    <script src="/widgets.js" type="text/javascript"></script>
    <title>Welcome to Joe's Widget Shop</title>
  </head>
  <body>
    <div id="content">
<h1>Joe's Widget Shop</h1>
<p>Welcome to Joe's Widget Shop, serving the greater Gotham, Idaho area!</p>
    </div>
    <div id="footer">Copyright 2010 Joe's Widget Shop</div>
  </body>
</html>
# end of index.html

# about.html:
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
  <head>
    <link rel="stylesheet" href="/widget.css">
    <script src="/widgets.js" type="text/javascript"></script>
    <title>About Joe's Widget Shop</title>
  </head>
  <body>
    <div id="content">
<h1>About Joe's Widget Shop</h1>
<p>One warm summer night in '76, Joe had a vision: A clean and well-lit shop,
with widgets as far as the eye could see. It wasn't long before...</p>
    </div>
    <div id="footer">Copyright 2010 Joe's Widget Shop</div>
  </body>
</html>
# end of about.html

MAGIC VARIABLES

In your variable definitions file (the one passed to the -v option),
variables whose name starts with "gs_" are reserved for magical
purposes.  There two defined right now: gs_path and gs_pagevars.

* GS_PATH

The value of gs_path is the path to the template being rendered,
relative to the base.  You're not allowed to set gs_path; instead, it
will just be available in your templates.

So suppose that in your source directory, the file
introspective/filename.html contains this content:

# introspective/filename.html:
This template is at {{gs_path}}.
# end of introspective/filename.html

That will render as:
# rendered introspective/filename.html:
This template is at introspective/filename.html.
# end of rendered introspective/filename.html

Note that gs_path points to the final path of the file.  So if you
include or extend a template, gs_path is not the path of that included
template:

# _/base.html:
<h1>{{gs_path}}</h1>
{% block content %}{% endblock %}
# end of _/base.html

# target.html:
{% extends "_/base.html" %}
{% block content %}
<p>Hello.</p>
{% endblock %}
# end of target.html

Then target.html will render like this:

# render of target.html:
<h1>target.html</h1>
<p>Hello.</p>
# end render of target.html

* GS_PAGEVARS

You define gs_pagevars in your variable definition file.  It lets you
specify customized or alternative variables for specific pages.  Its
type is a dictionary (associative array), mapping strings to other
dictionaries.

Each key of gs_pagevar is a path to a template: the value of gs_path
for that page, essentially.  The value is a dictionary mapping
variable names (strings) to any type of value.  These variables are
then added to the template parameter set for only that page.

Here's an example.  Suppose that you have three pages, one.html,
two.html, and another-one.html.  The first and last have basically the
same content.  You don't want to get some kind of duplicate content
penalty in search engine results, so you want to indicate that the
former is the canonical version, using the "rel=canonical" meta tag.
In vars.py, you can define this:

# vars.py:
gs_pagevars = {
  'another-one.html' : { 'canonical' : 'http://example.com/one.html'},
}
# end vars.py

Suppose that all these HTML files inherit from a common "_/base.html"
template.  You can place this text in the <head> element:

# begin
{% if gs_pagevars.canonical %}
<link rel="canonical" href="{{gs_pagevars.canonical}}"/>
{% endif %}
# end

Then another-one.html will render with the this line in its head element:
# begin
<link rel="canonical" href="http://example.com/one.html"/>
# end

This line will be absent from the other templates.  Later, if you want
to add a new page "another-two.html" with a similar relationship, you
can just add a line to gs_pagevars:

# vars.py:
gs_pagevars = {
  'another-one.html' : { 'canonical' : 'http://example.com/one.html'},
  'another-two.html' : { 'canonical' : 'http://example.com/two.html'},
}
# end vars.py

LICENSE

Genstatic is licensed under the GPL version 3.

This source code is copyright 2009-2010 Aaron Maxwell.  All rights reserved.

INSTALL

Genstatic depends on Python 2.6 or later, and Django 1.2.  Install
both of these on your system, and make sure Django is in the python
path. (i.e. is one of the directories in the PYTHONPATH environment
variable).

THANKS

Thanks to Mobile Web Up (http://mobilewebup.com) for sponsoring
genstatic development.  If you or someone you know needs to upgrade a
website for mobile phones, or start a mobile marketing campaign,
contact Mobile Web Up today.

APPENDIX A: PYTHON VARIABLES

To make varibles available to genstatic (with the -v option), you'll
have to create a small Python file that defines some variables and
their values.  It's not too hard.

The official Python tutorial has a section, "Using Python as a
calculator", that is very nice, and can teach you all you'll need to
know.  Read it here:
http://docs.python.org/tutorial/introduction.html#using-python-as-a-calculator

Short version: you can just create a file named "something.py", and
put in some lines like this:

name = value

... where 'name' is a variable name, and 'value' is what the variable
is set to.  'value' can be a whole number (integer), a floating-point
number, or a string (surrounded by either single or double quotes).
Single and double quotes are equivalent in python, and they both
interpolate.  So you'll need to backslash-escape any special
characters.

Python has sequence and collection types as well - lists and
dictionaries (associative arrays), etc.  You can certainly define
variables of these types, even write arbitrary Python code to generate
what you want.  Dig deeper into the Python tutorial above if you'd
like to know how to do this.
