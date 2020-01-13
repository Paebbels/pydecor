Why PyDecor?
############

.. rubric:: It's easy!

With PyDecor, you can go from this:

.. code-block:: python

   from functools import wraps
   from flask import request
   from werkzeug.exceptions import Unauthorized
   from my_pkg.auth import authorize_request

   def auth_decorator(request=None):
       """Check the passed request for authentication"""

       def decorator(decorated):

           @wraps(decorated)
           def wrapper(*args, **kwargs):
               if not authorize_request(request):
                 raise Unauthorized('Not authorized!')
               return decorated(*args, **kwargs)
           return wrapper

       return decorated

   @auth_decorator(request=requst)
   def some_view():
       return 'Hello, World!'

to this:

.. code-block:: python

   from flask import request
   from pydecor import before
   from werkzeug.exceptions import Unauthorized
   from my_pkg.auth import authorize_request

   def check_auth(request=request):
       """Ensure the request is authorized"""
       if not authorize_request(request):
         raise Unauthorized('Not authorized!')

   @before(check_auth, request=request)
   def some_view():
       return 'Hello, world!'

Not only is it less code, but you don't have to remember decorator
syntax or mess with nested functions. Full disclosure, I had to look
up a decorator sample to be sure I got the first example's syntax right,
and I just spent two weeks writing a decorator library.

.. rubric:: It's fast!

PyDecor aims to make your life easier, not slower. The decoration machinery
is designed to be as efficient as is reasonable, and contributions to
speed things up are always welcome.

.. rubric:: Implicit Method Decoration!

Getting a decorator to "roll down" to methods when applied to a class is
a complicated business, but all of PyDecor's decorators provide it for
free, so rather than writing:

.. code-block:: python

   from pydecor import log_call

   class FullyLoggedClass(object):

       @log_call(level='debug')
       def some_function(self, *args, **kwargs):
           return args, kwargs

       @log_call(level='debug')
       def another_function(self, *args, **kwargs):
           return None

       ...

You can just write:

.. code-block:: python

   from pydecor import log_call

   @log_call(level='debug')
   class FullyLoggedClass(object):

       def some_function(self, *args, **kwargs):
           return args, kwargs

       def another_function(self, *args, **kwargs):
           return None

       ...

PyDecor ignores special methods (like ``__init__``) so as not to interfere
with deep Python magic. By default, it works on any methods of a class,
including instance, class and static methods. It also ensures that class
attributes are preserved after decoration, so your class references
continue to behave as expected.

.. rubric:: Consistent Method Decoration!

Whether you're decorating a class, an instance method, a class method, or
a static method, you can use the same passed function. ``self`` and ``cls``
variables are stripped out of the method parameters passed to the provided
callable, so your functions don't need to care about where they're used.

.. rubric:: Lots of Tests!

Seriously. Don't believe me? Just look. We've got the best tests. Just
phenomenal.
