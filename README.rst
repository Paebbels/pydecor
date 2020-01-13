PyDecor
=======

.. image:: https://travis-ci.org/mplanchard/pydecor.svg?branch=master
   :target: https://travis-ci.org/mplanchard/pydecor

.. image:: https://readthedocs.org/projects/pydecor/badge/?version=latest
    :target: https://pydecor.readthedocs.io/

Easy-peasy Python decorators!

* GitHub: https://github.com/mplanchard/pydecor
* PyPI: https://pypi.python.org/pypi/pydecor
* Docs: https://pydecor.readthedocs.io/
* Contact: ``msplanchard`` ``@`` ``gmail`` or @msplanchard on Twitter


Decorators are great, but they're hard to write, especially if you want
to include arguments to your decorators, or use your decorators on
class methods as well as functions. I know that, no matter how many I write,
I still find myself looking up the syntax every time. And that's just for
simple function decorators. Getting decorators to work consistently at the
class and method level is a whole 'nother barrel of worms.

PyDecor aims to make function decoration easy and straightforward, so that
developers can stop worrying about closures and syntax in triply nested
functions and instead get down to decorating!


IMPORTANT: Upcoming Backwards Incompatible Changes
--------------------------------------------------

Version 2.0.0 will make some changes to the call signatures for functions
passed to ``@before``, ``@after``, ``@instead``, ``@decorate``, and
``construct_decorator``, as well as to the call signatures to the
decorators themselves.

Specifically, rather than defaulting the call signature to some subset
of decorated function args, kwargs, result, and the decorated function
itself and allowing overrides with keyword arguments
to the decorator like ``pass_params``, all functions passed to ``@before``,
``@after``, and ``@instead`` will receive an immutable ``Decorated``
object, which will have ``args``, ``kwargs``, ``wrapped``, and ``result``
attributes, and which will support direct calls as though it were the
decorated function/method/class. The aim of this is to make writing functions
to pass to the decorators more intuitive, but it will require some minor
re-writing of passed functions.

You can experiment with this syntax and prepare for the cut-over right away
by passing ``_use_future_syntax=True`` to any of your generic decorators
(``@after``, ``@before``, etc.) or to ``construct_decorator``. See the below
snippet to illustrate basic use of the new ``Decorated`` object:

.. code-block:: python

   from pydecor import after, Decorated

   def after_func(decorated: Decorated, extra_kwarg=None):
       """A function to be called after the decorated function"""
       assert decorated.args == ('foo', )
       assert decorated.kwargs == {'bar': 'bar'}
       assert decorated.result == 'baz'
       assert extra_kwarg == 'extra_kwarg'


   @after(after_func, extra_kwarg='extra_kwarg')
   def some_function('foo', bar='bar'):
       """A function that returns 'baz'"""
       return 'baz'

All of the builtin non-generic decorators (``@memoize``, ``@intercept``,
and ``@log_call``) are already using the future syntax, so feel free
to look at those for more examples.

See the API docs for more information.


Details
-------




Prête-à-porter (ready-to-wear)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* ``intercept`` - catch the specified exception and optionally re-raise and/or
  call a provided callback to handle the exception
* ``log_call`` - automatically log the decorated function's call signature and
  results
* ``memoize`` - memoize a function's call and return values for re-use. Can
  use any cache in ``pydecor.caches``, which all have options for automatic
  pruning to keep the memoization cache from growing too large.

**More to come!!** See Roadmap_ for more details on upcoming features


Caches
******

Three caches are provided with ``pydecor``. These are designed to be passed
to the ``@memoization`` decorator if you want to use something other than
the default ``LRUCache``, but they are perfectly functional for use elesewhere.

All caches implement the standard dictionary interface.


LRUCache
~~~~~~~~

A least-recently-used cache. Both getting and setting of key/value pairs
results in their having been considered most-recently-used. When the cache
reaches the specified ``max_size``, least-recently-used items are discarded.

FIFOCache
~~~~~~~~~

A first-in, first-out cache. When the cache reaches the specified ``max_size``,
the first item that was inserted is discarded, then the second, and so on.

TimedCache
~~~~~~~~~~

A cache whose entries expire. If a ``max_age`` is specified, any entries older
than the ``max_age`` (in seconds) will be considered invalid, and will be
removed upon access.


Stacking
********

Generic and convenience decorators may be stacked! You can stack multiple
of the same decorator, or you can mix and match. Some gotchas are listed
below.

Generally, staciking works just as you might expect, but some care must be
taken when using the ``@instead`` decorator, or ``@intercept``, which
uses ``@instead`` under the hood.

Just remember that ``@instead`` replaces everything that comes before. So,
if long as ``@instead`` calls the decorated function, it's okay to stack it.
In these cases, it will be called *before* any decorators specified below
it, and those decorators will be executed when it calls the decorated function.
``@intercept`` behaves this way.

If an ``@instead`` decorator does *not* call the decorated function and
instead replaces it entirely, it **must** be specified first (at the bottom
of the stacked decorator pile), otherwise the decorators below it will not
execute.

For ``@before`` and ``@after``, it doesn't matter in what order the decorators
are specified. ``@before`` is always called first, and then ``@after``.


Class Decoration
****************

Class decoration is difficult, but PyDecor aims to make it as easy and
intuitive as possible!

By default, decorating a class applies that decorator to all of that class'
methods (instance, class, and static). The decoration applies to class and
static methods whether they are referenced via an instance or via a class
reference. "Extras" specified at the class level persist across calls to
different methods, allowing for things like a class level memoization
dictionary (there's a very basic test in the test suite
that demonstrates this pattern, and a convenient memoization decorator
is scheduled for the next release!).

If you'd prefer that the decorator not apply to class and static methods,
set the ``instance_methods_only=True`` when decorating the class.

If you want to decorate the class itself, and *not* its methods, keep in
mind that the decorator will be triggered when the class is instantiated,
and that, if the decorator replaces or alters the return, that return will
replace the instantiated class. With those caveats in mind, setting
``implicit_method_decoration=False`` when decorating a class enables that
funcitonality.

.. note::

    Class decoration, and in particular the decoration of class and static
    methods, is accomplished through some pretty deep, complicated magic.
    The test suite has a lot of tests trying to make sure that everything
    works as expected, but please report any bugs you find so that I
    can resolve them!


Method Decoration
*****************

Decorators can be applied to static, class, or instance methods directly, as
well. If combined with ``@staticmethod`` or ``@classmethod`` decorators,
those decorators should always be at the "top" of the decorator stack
(furthest from the function).

When decorating instance methods, ``self`` is removed from the parameters
passed to the provided callable.

When decorating class methods, ``cls`` is removed from the parameters passed
to the provided callable.

Currently, the class and instance references *do not* have to be named
``"cls"`` and ``"self"``, respectively, in order to be removed. However,
this is not guaranteed for future releases, so try to keep your naming
standard if you can (just FYI, ``"self"`` is the more likely of the two to
wind up being required).

Examples
--------

Below are some examples for the generic and standard decorators. Please
check out the API Docs for more information, and also check out the
convenience decorators, which are all implemented using the
``before``, ``after``, and ``instead`` decorators from this library.

Update a Function's Args or Kwargs
**********************************

Functions passed to ``@before`` can either return None, in which case nothing
happens to the decorated functions parameters, or they can return a tuple
of args (as a tuple) and kwargs (as a dict), in which case those parameters
are used in the decorated function. In this example, we sillify a very
serious function.

.. note::
    Because kwargs are mutable, they can be updated even if the function
    passed to before doesn't return anything.

.. code:: python

    from pydecor import before

    def spamify_func(args, kwargs):
        """Mess with the function arguments"""
        args = tuple(['spam' for _ in args])
        kwargs = {k: 'spam' for k in kwargs}
        return args, kwargs


    @before(spamify_func, pass_params=True)
    def serious_function(serious_string, serious_kwarg='serious'):
        """A very serious function"""
        print('A serious arg: {}'.format(serious_string))
        print('A serious kwarg: {}'.format(serious_kwarg))

    serious_function('Politics', serious_kwarg='Religion')

The output?

.. code::

    A serious arg: spam
    A serious kwarg: spam

Do Something with a Function's Return Value
*******************************************

Functions passed to ``@after`` receive the decorated function's return value
by default. If ``@after`` returns None, the return value is sent back
unchanged. However, if ``@after`` returns something, its return value is
sent back as the return value of the function.

In this example, we ensure that a function's return value has been thoroughly
spammified.

.. code:: python

    from pydecor import after

    def spamify_return(result):
        """Spamify the result of a function"""
        return 'spam-spam-spam-spam-{}-spam-spam-spam-spam'.format(result)


    @after(spamify_return)
    def unspammed_function():
        """Return a non-spammy value"""
        return 'beef'

    print(unspammed_function())

The output?

.. code::

    spam-spam-spam-spam-beef-spam-spam-spam-spam


Do Something Instead of a Function
**********************************

Functions passed to ``@instead`` by default receive the args and kwargs of
the decorated function, along with a reference to that function. But, they
don't *have* to receive anything. Maybe you want to skip a function when
a certain condition is True, but you don't want to use ``pytest.skipif``,
because ``pytest`` can't be a dependency of your production code for
whatever reason.


.. code:: python

    from pydecor import instead

    def skip(args, kwargs, decorated, when=False):
        if when:
            pass
        else:
            return decorated(*args, **kwargs)


    @instead(skip, when=True)
    def uncalled_function():
        print("You won't see me (you won't see me)")


    uncalled_function()

The output?

(There is no output, because the function was skipped)


Automatically Log Function Calls and Results
********************************************

Maybe you want to make sure your functions get logged without having to
bother with the logging boilerplate each time. ``@log_call`` tries to
automatically get a logging instance corresponding to the module
in which the decoration occurs (in the same way as if you made a call
to ``logging.getLogger(__name__)``, or you can pass it your own, fancy,
custom, spoiler-bedecked logger instance.

.. code:: python

    from logging import getLogger, StreamHandler
    from sys import stdout

    from pydecor import log_call


    # We're just getting a logger here so we can see the output. This isn't
    # actually necessary for @log_call to work!
    log = getLogger(__name__)
    log.setLevel('DEBUG')
    log.addHandler(StreamHandler(stdout))


    @log_call()
    def get_schwifty(*args, **kwargs):
        """Get schwifty in heeeeere"""
        return "Gettin' Schwifty"


    get_schwifty('wubba', 'lubba', dub='dub')


And the output?

.. code::

    get_schwifty(*('wubba', 'lubba'), **{'dub': 'dub'}) -> Gettin' Schwifty


Intercept an Exception and Re-raise a Custom One
************************************************

Are you a put-upon library developer tired of constantly having to re-raise
custom exceptions so that users of your library can have one nice try/except
looking for your base exception? Let's make that easier:

.. code:: python

    from pydecor import intercept


    class BetterException(Exception):
        """Much better than all those other exceptions"""


    @intercept(catch=RuntimeError, reraise=BetterException)
    def sometimes_i_error(val):
        """Sometimes, this function raises an exception"""
        if val > 5:
            raise RuntimeError('This value is too big!')


    for i in range(7):
        sometimes_i_error(i)


The output?

.. code::

    Traceback (most recent call last):
      File "/Users/Nautilus/Library/Preferences/PyCharm2017.1/scratches/scratch_1.py", line 88, in <module>
        sometimes_i_error(i)
      File "/Users/Nautilus/Documents/Programming/pydecor/pydecor/decorators.py", line 389, in wrapper
        return fn(**fkwargs)
      File "/Users/Nautilus/Documents/Programming/pydecor/pydecor/functions.py", line 58, in intercept
        raise_from(new_exc, context)
      File "<string>", line 2, in raise_from
    __main__.BetterException: This value is too big!


Intercept an Exception, Do Something, and Re-raise the Original
***************************************************************

Maybe you don't *want* to raise a custom exception. Maybe the original
one was just fine. All you want to do is print a special message before
re-raising the original exception. PyDecor has you covered:

.. code:: python

    from pydecor import intercept


    def print_exception(exc):
        """Make sure stdout knows about our exceptions"""
        print('Houston, we have a problem: {}'.format(exc))


    @intercept(catch=Exception, handler=print_exception, reraise=True)
    def assert_false():
        """All I do is assert that False is True"""
        assert False, 'Turns out, False is not True'


    assert_false()

And the output:

.. code::

    Houston, we have a problem: Turns out, False is not True
    Traceback (most recent call last):
      File "/Users/Nautilus/Library/Preferences/PyCharm2017.1/scratches/scratch_1.py", line 105, in <module>
        assert_false()
      File "/Users/Nautilus/Documents/Programming/pydecor/pydecor/decorators.py", line 389, in wrapper
        return fn(**fkwargs)
      File "/Users/Nautilus/Documents/Programming/pydecor/pydecor/functions.py", line 49, in intercept
        return decorated(*decorated_args, **decorated_kwargs)
      File "/Users/Nautilus/Library/Preferences/PyCharm2017.1/scratches/scratch_1.py", line 102, in assert_false
        assert False, 'Turns out, False is not True'
    AssertionError: Turns out, False is not True


Intercept an Exception, Handle, and Be Done with It
***************************************************

Sometimes an exception isn't the end of the world, and it doesn't need to
bubble up to the top of your application. In these cases, maybe just handle
it and don't re-raise:

.. code:: python

    from pydecor import intercept


    def let_us_know_it_happened(exc):
        """Just let us know an exception happened (if we are reading stdout)"""
        print('This non-critical exception happened: {}'.format(exc))


    @intercept(catch=ValueError, handler=let_us_know_it_happened)
    def resilient_function(val):
        """I am so resilient!"""
        val = int(val)
        print('If I get here, I have an integer: {}'.format(val))


    resilient_function('50')
    resilient_function('foo')

Output:

.. code::

    If I get here, I have an integer: 50
    This non-critical exception happened: invalid literal for int() with base 10: 'foo'

Note that the function does *not* continue running after the exception is
handled. Use this for short-circuiting under certain conditions rather
than for instituting a ``try/except:pass`` block. Maybe one day I'll figure
out how to make this work like that, but as it stands, the decorator surrounds
the entire function, so it does not provide that fine-grained level of control.


Roadmap
-------

1.2.0
*****

More Prête-à-porter Decorators
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* ``export`` - add the decorated item to ``__all__``
* ``skipif`` - similar to py.test's decorator, skip the function if a
  provided condition is True

Let me know if you've got any idea for other decorators that would
be nice to have!


Typing Stubfiles
~~~~~~~~~~~~~~~~

Right now type hints are provided via rst-style docstring specification.
Although this format is supported by PyCharm, it does not conform to the
type-hinting standard defined in `PEP 484`_.

In order to better conform with the new standard (and to remain compatible
with Python 2.7), stubfiles will be added for the ``1.1.0`` release,
and docstring hints will be removed so that contributors don't have
to adjust type specifications in two places.

Build-process Updates
~~~~~~~~~~~~~~~~~~~~~

A more automated build process, because remembering all the steps to push a
new version is a pain. This is marked as scheduled for a patch release,
because it does not affect users at all, so a minor version bump would
lead people on to thinking that some new functionality had been added, when
it hadn't.


2.0.0
*****

* Use of immutable ``Decorated`` object to pass information about the
  deprecated function
* Deprecation of ``pass_params``, ``pass_kwargs``, ``pass_decorated``,
  ``pass_result``, ``unapck_extras``, and ``extras_key`` keyword
  arguments to all decorators.
* Better organization of documentation


Contributing
------------

Contributions are welcome! If you find a bug or if something doesn't
work the way you think it should, please `raise an issue <issues_>`_.
If you know how to fix the bug, please `open a PR! <prs_>`_

I absolutely welcome any level of contribution. If you think the docs
could be better, or if you've found a typo, please open up a PR to improve
and/or fix them.

