PyOtherSide Developer Guide
===========================

*PyOtherSide* is a Qt 5 QML Plugin that provides access to a Python 3
interpreter from QML. It was designed with mobile devices in mind, where
high-framerate touch interfaces are common, and where the user usually
interfaces only with one application at a time via a touchscreen. As such, it
is important to never block the UI thread, so that the user can always continue
to use the interface, even when the backend is processing, downloading or
calculating something in the background.

At its core, PyOtherSide is basically a simple layer that converts Qt (QML)
objects to Python objects and vice versa, with focus on asynchronous events
and continuation-passing style function calls.

While PyOtherSide historically also worked with Python 2.x and Qt 4.x, its
focus now lies on Python 3.x and Qt 5. Python 3 has been out for several years,
and offers some nice language features and clean-ups, while Qt 5 supports most
mobile platforms well, and has an improved QML engine and a faster renderer (Qt
Scene Graph) compared to Qt 4.


QML API
=======

This section describes the QML API exposed by the *PyOtherSide* QML Plugin.
The current QML API version of PyOtherSide is 1.0. When new features are
introduced, the API version will be bumped and documented here.

Import Versions
---------------

**io.thp.pyotherside 1.0** (since: 0.1.0)
    Initial API release.

QML ``Python`` Element
----------------------

The ``Python`` element exposes a Python interpreter in a QML file. In
PyOtherSide 1.0, if multiple Python elements are instantiated, they will share
the same underlying Python interpreter, so Python module-global state will be
shared between all Python elements.

To use the ``Python`` element in a QML file, you have to import the plugin using:

.. code-block:: javascript

    import io.thp.pyotherside 1.0

Signals
```````

**received(var data)**
    Default event handler for ``pyotherside.send()``
    if no other event handler was set.

**error(string traceback)**
    Error handler for errors from Python.

Methods
```````

To configure event handlers for events from Python, you can use
the ``setHandler(event, callback)`` method:

**setHandler(string event, callable callback)**
    Set the handler for events sent with ``pyotherside.send()``.

Importing modules is then done by optionally adding an import
path and then importing the module asynchronously:

**addImportPath(string path)**
    Add a local filesystem path to Python's ``sys.path``.

**importModule(string name, callable callback)**
    Import a Python module.

Once modules are imported, Python function can be called on the
imported modules using:

**call(string func, args=[], function callback(result) {})**
    Call the Python function ``func`` with ``args`` asynchronously.
    If ``args`` is omitted, ``func`` will be called without arguments.
    If ``callback`` is a callable, it will be called with the Python
    function result as single argument when the call has succeeded.

For some of these methods, there also exist synchronous variants, but it is
highly recommended to use the asynchronous variants instead to avoid blocking
the QML UI thread:

**var evaluate(string expr)**
    Evaluate a Python expression synchronously.

**bool importModule_sync(string name)**
    Import a Python module. Returns ``true`` on success, ``false`` otherwise.

**var call_sync(string func, var args=[])**
    Call a Python function. Returns the return value of the Python function.

Python API
==========

PyOtherSide uses a normal Python 3.x interpreter for running your Python code.

The ``pyotherside`` module
--------------------------

When a module is imported in PyOtherSide, it will have access to a special
module called ``pyotherside`` in addition to all Python Standard Library modules
and Python modules in ``sys.path``:

.. code-block:: python

    import pyotherside

The module can be used to send events asynchronously (even from different threads)
to the QML layer, register a callback for doing clean-ups at application exit and
integrate with other QML-specific features of PyOtherSide.

Methods
```````

**pyotherside.send(event, *args)**
    Send an asynchronous event with name ``event`` with optional arguments ``args`` to QML.

**pyotherside.atexit(callback)**
    Register a ``callback`` to be called when the application is closing.

**pyotherside.set_image_provider(provider)**
    Set the QML `image provider`_ (``image://python/``).

.. _constants:

Constants
`````````

These constants are used in the return value of a `image provider`_ function:

**pyotherside.format_mono**
    Mono pixel format (``QImage::Format_Mono``).

**pyotherside.format_mono_lsb**
    Mono pixel format, LSB alignment (``QImage::Format_MonoLSB``).

**pyotherside.format_rgb32**
    32-bit RGB format (``QImage::Format_RGB32``).

**pyotherside.format_argb32**
    32-bit ARGB format (``QImage::Format_ARGB32``).

**pyotherside.format_rgb16**
    16-bit RGB format (``QImage::Format_RGB16``).

**pyotherside.format_rgb666**
    18bpp RGB666 format (``QImage::Format_RGB666``).

**pyotherside.format_rgb555**
    15bpp RGB555 format (``QImage::Format_RGB555``).

**pyotherside.format_rgb888**
    24-bit RGB format (``QImage::Format_RGB888``).

**pyotherside.format_rgb444**
    12bpp RGB format (``QImage::Format_RGB444``).

**pyotherside.format_data**
    Encoded image file data (e.g. PNG/JPEG data).


Data Type Mapping
=================

PyOtherSide will automatically convert Python data types to Qt data types
(which in turn will be converted to QML data types by the QML engine).
The following data types are supported and can be used to pass data
between Python and QML (and vice versa):

+------------+------------+-----------------------------+
| Python     | QML        | Remarks                     |
+============+============+=============================+
| bool       | bool       |                             |
+------------+------------+-----------------------------+
| int        | int        |                             |
+------------+------------+-----------------------------+
| float      | double     |                             |
+------------+------------+-----------------------------+
| str        | string     |                             |
+------------+------------+-----------------------------+
| list       | JS Array   |                             |
+------------+------------+-----------------------------+
| tuple      | JS Array   | JS Arrays are converted to  |
|            |            | lists, no tuples            |
+------------+------------+-----------------------------+
| dict       | JS Object  | Keys must be strings        |
+------------+------------+-----------------------------+

Trying to pass in other types than the ones listed here is undefined
behavior and will usually result in an error.

.. _image provider:

Image Provider
==============

A QML Image Provider can be registered from Python to load image
data (e.g. map tiles, diagrams, graphs or generated images) in
QML ``Image`` elements without resorting to saving/loading files.

An image provider has the following argument list and return values:

.. code-block:: python

    def image_provider(image_id, requested_size):
        ...
        return bytearray(pixels), (width, height), format

The parameters to the image provider functions are:

**image_id**
    The ID of the image URL (``image://python/<image_id>``).

**requested_size**
    The source size of the QML ``Image`` as tuple: ``(width, height)``.
    ``(-1, -1)`` if the source size is not set.

The image provider must return a tuple ``(data, size, format)``:

**data**
    A ``bytearray`` object containing the pixel data for the
    given size and the given format.

**size**
    A tuple ``(width, height)`` describing the size of the
    pixel data in pixels.

**format**
    The pixel format of ``data`` (see `constants`_), or
    ``pyotherside.format_data`` if ``data`` contains an
    encoded (PNG/JPEG) image instead of raw pixel data.

In order to register the image provider with PyOtherSide for use
as provider for ``image://python/`` URLs, the image provider function
needs to be passed to PyOtherSide:

.. code-block:: python

    import pyotherside

    def image_provider(image_id, requested_size):
        ...

    pyotherside.set_image_provider(image_provider)

Because Python modules are usually imported asynchronously, the image
provider will only be registered once the module registering the image
provider is successfully imported. You have to make sure that setting
the ``source`` property on a QML ``Image`` element only happens *after*
the image provider has been set (e.g. by setting the ``source`` property
in the callback function passed to ``importModule``).

Cookbook
========

This section contains code examples and best practices for combining Python and
QML.

Importing modules and calling functions asynchronously
------------------------------------------------------

In this example, we import the Python Standard Library module ``os``
and - when the module is imported - call the ``os.getcwd()`` function on it.
The result of the ``os.getcwd()`` function is then printed to the console
and ``os.chdir()`` is called with a single argument (``'/'``) - again, after
the ``os.chdir()`` function has returned, a message will be printed.

In this example, importing modules and calling functions are both done in
an asynchronous way - the QML/GUI thread will not block while these functions
execute. In fact, the ``Component.onCompleted`` code block will probably
finish before the ``os`` module has been imported in Python.

.. code-block:: javascript

    Python {
        Component.onCompleted: {
            importModule('os', function() {
                call('os.getcwd', [], function (result) {
                    console.log('Working directory: ' + result);
                    call('os.chdir', ['/'], function (result) {
                        console.log('Working directory changed.');
                    }););
                });
            });
        }
    }

While this `continuation-passing style`_ might look a like a little pyramid
due all the nesting and indentation at first, it makes sure your application's
UI is always responsive. The user will be able to interact with the GUI (e.g.
scroll and move around in the UI) while the Python code can process requests.

.. _Continuation-passing style: https://en.wikipedia.org/wiki/Continuation-passing_style

Evaluating Python expressions in QML
````````````````````````````````````

The ``evaluate()`` method on the ``Python`` object can be used to evaluate a
simple Python expression and return its result as JavaScript object:

.. code-block:: javascript

    Python {
        Component.onCompleted: {
            console.log('Squares: ' + evaluate('[x for x in range(10)]'));
        }
    }

Evaluating expressions is done synchronously, so make sure you only use it for
expressions that are not long-running calculations / operations.


Error handling in QML
---------------------

If an error happens in Python while calling functions, the traceback of the
error (or an error message in case the error happens in the PyOtherSide layer)
will be sent with the ``error`` signal of the ``Python`` element. During early
development, it's probably enough to just log the error to the console:

.. code-block:: javascript

    Python {
        // ...

        onError: console.log('Error: ' + traceback)
    }

Once your application grows, it might make sense to maybe show the error to the
user in a dialog box, message or notification in addition to or instead of using
``console.log()`` to print the error.


Handing asynchronous events from Python in QML
----------------------------------------------

Your Python code can send asynchronous events with optional data to the QML
layer using the ``pyotherside.send()`` function. You can call this function from
functions called from QML, but also from anywhere else - including threads that
you created in Python. The first parameter is mandatory, and must be a string
that identifies the event. Additional parameters are optional and can be of any
data type that PyOtherSide supports:

.. code-block:: python

    import pyotherside

    pyotherside.send('new-entries', 100, 123)

If you do not add a special handler on the ``Python`` object, such events would
be handled by the ``onReceived`` signal in QML - its ``data`` parameter contains
the event name and all arguments in a list:

.. code-block:: javascript

    Python {
        // ..

        onReceived: console.log('Event: ' + data)
    }

Usually, you want to install a handler for such events. If you have e.g. the
``'new-entries'`` event like shown above (with two numeric parameters that we
will call ``first`` and ``last`` for this example), you might want to define a
simple handler function that will process this event:

.. code-block:: javascript

    Python {
        // ..

        Component.onCompleted: {
            setHandler('new-entries', function (first, last) {
                console.log('New entries from ' + first + ' to ' + last);
            });
        }
    }

Once a handler for a given event is defined, the ``onReceived`` signal will not
be emitted anymore. If you need to unset a handler for a given event, you can
use ``setHandler('event', undefined)`` to do so.

In some cases, it might be useful to not install a handler function directly, but
turn the ``pyotherside.send()`` call into a new signal on the ``Python`` object.
As there is no easy way for PyOtherSide to determine the names of the arguments
of the event, you have to define and hook up these signals manually. The upside
of having to define the signals this way is that all signals will be nicely
documented in your QML file for future reference:

.. code-block:: javascript

    Python {
        signal updated()
        signal newEntries(int first, int last)
        signal entryRenamed(int index, string name)

        Component.onCompleted: {
            setHandler('updated', updated);
            setHandler('new-entries', newEntries);
            setHandler('entry-renamed', entryRenamed);
        }
    }

With this setup, you can now emit these signals from the ``Python`` object by
using ``pyotherside.send()`` in your Python code:

.. code-block:: python

    pyotherside.send('updated')
    pyotherside.send('new-entries', 20, 30)
    pyotherside.send('entry-renamed', 11, 'Hello World')


Loading ``ListModel`` data from Python
--------------------------------------

Most of the time a PyOtherSide QML application will display some data stored
somewhere and retrieved or generated with Python. The easiest way to do this is
to return a list-of-dicts in your Python function:

**listmodel.py**

.. code-block:: python

    def get_data():
        return [
            {'name': 'Alpha', 'team': 'red'},
            {'name': 'Beta', 'team': 'blue'},
            {'name': 'Gamma', 'team': 'green'},
            {'name': 'Delta', 'team': 'yellow'},
            {'name': 'Epsilon', 'team': 'orange'},
        ]

Of course, the function could do other things (such as doing web requests, querying
databases, etc..) - as long as it returns a list-olf-dicts, it will be fine (if you
are using a generator that yields dicts, just wrap the generator with ``list()``).
Using this function from QML is straightforward:

**listmodel.qml**

.. code-block:: javascript

    import QtQuick 2.0
    import io.thp.pyotherside 1.0

    Rectangle {
        color: 'black'
        width: 400
        height: 400

        ListView {
            anchors.fill: parent

            model: ListModel {
                id: listModel
            }

            delegate: Text {
                // Both "name" and "team" are taken from the model
                text: name
                color: team
            }
        }

        Python {
            id: py

            Component.onCompleted: {
                // Add the directory of this .qml file to the search path
                addImportPath(Qt.resolvedUrl('.').substr('file://'.length));

                // Import the main module and load the data
                importModule('listmodel', function () {
                    py.call('listmodel.get_data', [], function(result) {
                        // Load the received data into the list model
                        for (var i=0; i<result.length; i++) {
                            listModel.append(result[i]);
                        }
                    });
                });
            }
        }
    }

Instead of passing a list-of-dicts, it is of course also possible to send
new list items via ``pyotherside.send()``, one item at a time, and append
them to the list model that way.

Rendering RGBA image data in Python
-----------------------------------

.. image:: images/image_provider_example.png

This example uses the `image provider`_ feature of PyOtherSide to
render RGB image data in Python and display the rendered data in
QML using a normal QtQuick 2.0 ``Image`` element:

**imageprovider.py**

.. code-block:: python

    import pyotherside
    import math

    def render(image_id, requested_size):
        print('image_id: "{image_id}", size: {requested_size}'.format(**locals()))

        # width and height will be -1 if not set in QML
        if requested_size == (-1, -1):
            requested_size = (300, 300)

        width, height = requested_size

        # center for circle
        cx, cy = width/2, 10

        pixels = []
        for y in range(height):
            for x in range(width):
                pixels.extend(reversed([
                    255, # alpha
                    int(10 + 10 * ((x - y * 0.5) % 20)), # red
                    20 + 10 * (y % 20), # green
                    int(255 * abs(math.sin(0.3*math.sqrt((cx-x)**2 + (cy-y)**2)))) # blue
                ]))
        return bytearray(pixels), (width, height), pyotherside.format_argb32

    pyotherside.set_image_provider(render)

This module can now be imported in QML and used as ``source`` in the QML
``Image`` element:

**imageprovider.qml**

.. code-block:: javascript

    import QtQuick 2.0
    import io.thp.pyotherside 1.0

    Image {
        id: image
        width: 300
        height: 300

        Python {
            Component.onCompleted: {
                // Add the directory of this .qml file to the search path
                addImportPath(Qt.resolvedUrl('.').substr('file://'.length));

                importModule('imageprovider', function () {
                    image.source = 'image://python/image-id-passed-from-qml';
                });
            }

            onError: console.log('Python error: ' + traceback)
        }
    }



Search
======

* :ref:`search`
