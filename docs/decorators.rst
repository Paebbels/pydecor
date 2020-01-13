Decorators
##########

This package provides generic decorators, which can be used with any
function to provide extra utility to decorated resources, as well
as prête-à-porter (ready-to-wear) decorators for immediate use.

While the information below is enough to get you started, I highly
recommend checking out the `decorator module docs`_ to see all the
options and details for the various decorators!

Generic Decorators
******************

* ``@before`` - run a callable before the decorated function executes

  * by default called with no arguments other than extras

* ``@after`` - run a callable after the decorated function executes

  * by default called with the result of the decorated function and any
    extras

* ``@instead`` - run a callable in place of the decorated function

  * by default called with the args and kwargs to the decorated function,
    along with a reference to the function itself

* ``@decorate`` - specify multiple callables to be run before, after, and/or
  instead of the decorated function

  * callables passed to ``decorate``'s ``before``, ``after``, or ``instead``
    keyword arguments will be called with the same default function signature
    as described for the individual decorators, above. Extras will be
    passed to all provided callables

* ``construct_decorator`` - specify functions to be run ``before``, ``after``,
  or ``instead``. Returns a reusable generator.

  * in addition to ``before``, ``after``, and ``instead``, which receive
    callables, ``before_opts``, ``after_opts``, and ``instead_opts`` dicts
    may be passed to ``construct_decorator``, and they will apply in the same
    way as their respective decorator parameters

Every generic decorator takes any number of keyword arguments, which will be
passed directly into the provided callable, unless ``unpack_extras`` is False
(see below), so, running the code below prints "red":

.. code-block:: python

   from pydecor import before

   def before_func(label=None):
       print(label)

   @before(before_func, label='red')
   def red_function():
       pass

   red_function()

Every generic decorator takes the following keyword arguments:

* ``pass_params`` - if True, passes the args and kwargs, as a tuple and
  a dict, respectively, from the decorated function to the provided callable
* ``pass_decorated`` - if True, passes a reference to the decorated function
  to the provided callable
* ``implicit_method_decoration`` - if True, decorating a class implies
  decorating all of its methods. **Caution:** you should probably leave this
  on unless you know what you are doing.
* ``instance_methods_only`` - if True, only instance methods (not class or
  static methods) will be automatically decorated when
  ``implicit_method_decoration`` is True
* ``unpack_extras`` - if True, extras are unpacked into the provided callable.
  If False, extras are placed into a dictionary on ``extras_key``, which
  is passed into the provided callable.
* ``extras_key`` - the keyword to use when passing extras into the provided
  callable if ``unpack_extras`` is False
* ``_use_future_syntax`` - See the note at the top on backwards incompatible
  changes in version 2.0.0.

The ``construct_decorator`` function can be used to combine ``@before``,
``@after``, and ``@instead`` calls into one decorator, without having to
worry about unintended stacking effects. Let's make a
decorator that announces when we're starting an exiting a function:

.. code-block:: python

   from pydecor import construct_decorator

   def before_func(decorated_func):
       print('Starting decorated function '
             '"{}"'.format(decorated_func.__name__))

   def after_func(decorated_result, decorated_func):
       print('"{}" gave result "{}"'.format(
           decorated_func.__name__, decorated_result
       ))

   my_decorator = construct_decorator(
       before=before_func,
       after=after_func,
       before_opts={'pass_decorated': True},
       after_opts={'pass_decorated': True},
   )

   @my_decorator()
   def this_function_returns_nothing():
       return 'nothing'

And the output?

.. code-block::

   Starting decorated function "this_function_returns_nothing"
   "this_function_returns_nothing" gave result "nothing"


Maybe a more realistic example would be useful. Let's say we want to add
headers to a Flask response.

.. code-block:: python


   from flask import Flask, Response, make_response
   from pydecor import construct_decorator


   def _set_app_json_header(response):
       # Ensure the response is a Response object, even if a tuple was
       # returned by the view function.
       response = make_response(response)
       response.headers.set('Content-Type', 'application/json')
       return response


   application_json = construct_decorator(after=_set_app_json_header)


   # Now you can decorate any Flask view, and your headers will be set.

   app = Flask(__name__)

   # Note that you must decorate "before" (closer to) the function than the
   # app.route() decoration, because the route decorator must be called on
   # the "finalized" version of your function

   @app.route('/')
   @application_json()
   def root_view():
       return 'Hello, world!'

   client = app.test_client()
   response = app.get('/')

   print(response.headers)


The output?

.. code-block::

   Content-Type: application/json
   Content-Length: 13
