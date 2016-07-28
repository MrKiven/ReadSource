.. _index:

Gunicorn
=========

Gunicorn源码阅读
----------------

从哪里开始
----------

首先从gunicorn的 `setup.py` 开始吧.
里面的 `entry_points` 里这样:

.. code:: python

    entry_points="""
    [console_scripts]
    gunicorn=gunicorn.app.wsgiapp:run
    gunicorn_django=gunicorn.app.djangoapp:run
    gunicorn_paster=gunicorn.app.pasterapp:run

    [paste.server_runner]
    main=gunicorn.app.pasterapp:paste_server
    """

所以我们从最基础的 `wsgiapp` 看起.

WSGIApplication
^^^^^^^^^^^^^^^^

入口类就在这里:

.. code:: python

    def run():
        """Here is docstring"""
        from gunicorn.app.wsgiapp import WSGIApplication
        WSGIApplication("%(prog)s [OPTIONS] [APP_MODULE]").run()

然后我们可以看到 `WSGIApplication` 这个class, 其实代码也不长:

.. code:: python

    class WSGIApplication(Application):
        pass
