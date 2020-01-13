Quickstart
##########

Install ``pydecor``::

  pip install pydecor

Use one of the ready-to-wear decorators:

.. code:: python

    # Memoize a function

    from pydecor import memoize


    @memoize()
    def fibonacci(n):
        """Compute the given number of the fibonacci sequence"""
        if n < 2:
            return n
        return fibonacci(n - 2) + fibonacci(n - 1)

    print(fibonacci(150))


.. code:: python

    # Intercept an error and raise a different one

    from flask import Flask
    from pydecor import intercept
    from werkzeug.exceptions import InternalServerError


    app = Flask(__name__)


    @app.route('/')
    @intercept(catch=Exception, reraise=InternalServerError,
               err_msg='The server encountered an error rendering "some_view"')
    def some_view():
        """The root view"""
        assert False
        return 'Asserted False successfully!'


    client = app.test_client()
    response = client.get('/')

    assert response.status_code == 500
    assert 'some_view'.encode() in resp.data


Use a generic decorator to run your own functions ``@before``, ``@after``,
or ``@instead`` of another function, like in the following example,
which sets a User-Agent header on a Flask response:

.. code:: python

    from flask import Flask, make_response
    from pydecor import after


    app = Flask(__name__)


    def set_user_agent(view_result):
        """Sets the user-agent header on a result from a view"""
        resp = make_response(view_result)
        resp.headers.set('User-Agent', 'my_applicatoin')
        return resp


    @app.route('/')
    @after(set_user_agent)
    def index_view():
        return 'Hello, world!'


    client = app.test_client()
    response = client.get('/')
    assert response.headers.get('User-Agent') == 'my_application'


Or make your own decorator with ``construct_decorator``

.. code:: python

    from flask import request
    from pydecor import construct_decorator
    from werkzeug.exceptions import Unauthorized


    def check_auth(request):
        """Theoretically checks auth

        It goes without saying, but this is example code. You should
        not actually check auth this way!
        """
        if request.host != 'localhost':
            raise Unauthorized('locals only!')


    authed = construct_decorator(before=check_auth)


    app = Flask(__name__)


    @app.route('/')
    @authed(request=request)
    def some_view():
        """An authenticated view"""
        return 'This is sensitive data!'
