Silk
====

|Build Status|

Silk is a live profiling and inspection tool for the Django framework.
It primarily consists of:

-  Middleware for intercepting Requests/Responses
-  A wrapper around SQL execution for profiling of database queries
-  A context manager/decorator for profiling blocks of code and
   functions either manually or dynamically.
-  A user interface for inspection and visualisation of the above.

Documentation is below and a live demo is available at
http://mtford.co.uk/silk/, where the tool is actively profiling my
website and blog!

Contents
--------

-  `Requirements <#requirements>`__
-  `Features <#features>`__

   -  `Request Inspection <#request-inspection>`__
   -  `SQL Inspection <#sql-inspection>`__
   -  `Profiling <#profiling>`__

      -  `Decorator <#decorator>`__
      -  `Context Manager <#context-manager>`__

-  `Experimental Features <#experimental-features>`__

   -  `Dynamic Profiling <#dynamic-profiling>`__
   -  `Code Generation <#code-generation>`__

-  `Installation <#installation>`__

   -  `Existing Release <#existing-release>`__

      -  `Pip <#Pip>`__
      -  `Github Tag <#Github%20Tag>`__

   -  `Master <#master>`__

-  `Roadmap <#roadmap>`__

Requirements
------------

Silk has been tested with:

-  Django: 1.5, 1.6
-  Python: 2.7, 3.3, 3.4

I left out Django <1.5 due to the change in ``{% url %}`` syntax. Python
2.6 is missing ``collections.Counter``. Python 3.0 and 3.1 are not
available via Travis and also are missing the ``u'xyz'`` syntax for
unicode. Workarounds can likely be found for all these if there is any
demand. Django 1.7 is currently untested.

Features
--------

Request Inspection
~~~~~~~~~~~~~~~~~~

The Silk middleware intercepts and stores requests and responses and
stores them in the configured database. These requests can then be
filtered and inspecting using Silk's UI through the request overview:

It records things like:

-  Time taken
-  Num. queries
-  Time spent on queries
-  Request/Response headers
-  Request/Response bodies

and so on.

Further details on each request are also available by clicking the
relevant request:

SQL Inspection
~~~~~~~~~~~~~~

Silk also intercepts SQL queries that are generated by each request. We
can get a summary on things like the tables involved, number of joins
and execution time:

Before diving into the stack trace to figure out where this request is
coming from:

Profiling
~~~~~~~~~

Silk can also be used to profile random blocks of code/functions. It
provides a decorator and a context manager for this purpose.

For example:

::

    @silk_profile(name='View Blog Post')
    def post(request, post_id):
        p = Post.objects.get(pk=post_id)
        return render_to_response('post.html', {
            'post': p
        })

Whenever a blog post is viewed we get an entry within the Silk UI:

Silk profiling not only provides execution time, but also collects SQL
queries executed within the block in the same fashion as with requests:

Decorator
^^^^^^^^^

The silk decorator can be applied to both functions and methods

.. code:: python

    @silk_profile(name='View Blog Post')
    def post(request, post_id):
        p = Post.objects.get(pk=post_id)
        return render_to_response('post.html', {
            'post': p
        })

    class MyView(View):    
        @silk_profile(name='View Blog Post')
        def get(self, request):
            p = Post.objects.get(pk=post_id)
            return render_to_response('post.html', {
                'post': p
            })

Context Manager
^^^^^^^^^^^^^^^

Using a context manager means we can add additional context to the name
which can be useful for narrowing down slowness to particular database
records.

::

    def post(request, post_id):
        with silk_profile(name='View Blog Post #%d' % self.pk):
            p = Post.objects.get(pk=post_id)
            return render_to_response('post.html', {
                'post': p
            })

Experimental Features
---------------------

The below features are still in need of thorough testing and should be
considered experimental.

Dynamic Profiling
~~~~~~~~~~~~~~~~~

One of Silk's more interesting features is dynamic profiling. If for
example we wanted to profile a function in a dependency to which we only
have read-only access (e.g. system python libraries owned by root) we
can add the following to ``settings.py`` to apply a decorator at
runtime:

::

    SILKY_DYNAMIC_PROFILING = [{
        'module': 'path.to.module',
        'function': 'MyClass.bar'
    }]

which is roughly equivalent to:

::

    class MyClass(object):
        @silk_profile()
        def bar(self):
            pass

The below summarizes the possibilities:

.. code:: python


    """
    Dynamic function decorator
    """

    SILKY_DYNAMIC_PROFILING = [{
        'module': 'path.to.module',
        'function': 'foo'
    }]

    # ... is roughly equivalent to
    @silk_profile()
    def foo():
        pass

    """
    Dynamic method decorator
    """

    SILKY_DYNAMIC_PROFILING = [{
        'module': 'path.to.module',
        'function': 'MyClass.bar'
    }]

    # ... is roughly equivalent to
    class MyClass(object):

        @silk_profile()
        def bar(self):
            pass

    """
    Dynamic code block profiling
    """

    SILKY_DYNAMIC_PROFILING = [{
        'module': 'path.to.module',
        'function': 'foo',
        # Line numbers are relative to the function as opposed to the file in which it resides
        'start_line': 1,
        'end_line': 2,
        'name': 'Slow Foo'
    }]

    # ... is roughly equivalent to
    def foo():
        with silk_profile(name='Slow Foo'):
            print (1)
            print (2)
        print(3)
        print(4)

Note that dynamic profiling behaves in a similar fashion to that of the
python mock framework in that we modify the function in-place e.g:

.. code:: python

    """ my.module """
    from another.module import foo

    # ...do some stuff
    foo()
    # ...do some other stuff

,we would profile ``foo`` by dynamically decorating ``my.module.foo`` as
opposed to ``another.module.foo``:

.. code:: python

    SILKY_DYNAMIC_PROFILING = [{
        'module': 'my.module',
        'function': 'foo'
    }]

If we were to apply the dynamic profile to the functions source module
``another.module.foo`` **after** it has already been imported, no
profiling would be triggered.

Code Generation
~~~~~~~~~~~~~~~

Silk currently generates two bits of code per request:

Both are intended for use in replaying the request. The curl command can
be used to replay via command-line and the python code can be used
within a Django unit test or simply as a standalone script.

Installation
------------

Existing Release
~~~~~~~~~~~~~~~~

Pip
^^^

Silk is on PyPi. Install via pip (into your virtualenv) as follows:

::

    pip install django-silk

Github Tag
^^^^^^^^^^

Releases of Silk are available on
`github <https://github.com/mtford90/silk/releases>`__.

Once downloaded, run:

.. code:: bash

    pip install dist/django-silk-<version>.tar.gz

Then configure Silk in ``settings.py``:

.. code:: python

    MIDDLEWARE_CLASSES = (
        ...
        'silk.middleware.SilkyMiddleware',
    )

    INSTALLED_APPS = (
        ...
        'silk'
    )

and to your ``urls.py``:

.. code:: python

    urlpatterns += patterns('', url(r'^silk', include('silk.urls', namespace='silk')))

before running syncdb:

.. code:: python

    python manage.py syncdb

Silk will automatically begin interception of requests and you can
proceed to add profiling if required. The UI can be reached at
``/silk/``

Master
~~~~~~

First download the
`source <https://github.com/mtford90/silky/archive/master.zip>`__, unzip
and navigate via the terminal to the source directory. Then run:

.. code:: bash

    python package.py mas

You can either install via pip:

.. code:: bash

    pip install dist/django-silk-mas.tar.gz

or run setup.py:

.. code:: bash

    tar -xvf dist/django-silk-mas.tar.gz
    python dist/django-silk-mas/setup.py

You can then follow the steps in 'Existing Release' to include Silk in
your Django project.

Roadmap
-------

I would eventually like to use this in a production environment. There
are a number of things preventing that right now:

-  Effect on performance.

   -  For every SQL query executed, Silk executes another.

-  Questionable stability.
-  Space concerns.

   -  Silk would quickly generate a huge number of database records.
   -  Silk saves down both the request body and response body for each
      and every request handled by Django.

-  Security risks involved in making the Silk UI available.

   -  e.g. POST of password forms
   -  exposure of session cookies

.. |Build Status| image:: https://travis-ci.org/mtford90/silk.svg?branch=master
   :target: https://travis-ci.org/mtford90/silk
