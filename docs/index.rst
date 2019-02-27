.. title:: Tornado Web 服务

.. meta::
    :google-site-verification: g4bVhgwbVO1d9apCUsT-eKlApg31Cygbp8VGZY8Rf0g

|Tornado Web 服务|
====================

.. |Tornado Web Server| image:: tornado.png
    :alt: Tornado Web Server

`Tornado <http://www.tornadoweb.org>`_ 是一个Python Web框架和异步网络库, 最初是在 `FriendFeed <http://friendfeed.com>`_ 开发的
.  因为使用非阻塞网络I/O，Tornado可以支撑上万的网络连接， 使其成为
`长轮询 <http://en.wikipedia.org/wiki/Push_technology#Long_polling>`_,
`WebSockets <http://en.wikipedia.org/wiki/WebSocket>`_, 和其他需要与每个用户建立长期连接的应用程序的理想选择。

快速链接
-----------

* 当前版本: |version| (`download from PyPI <https://pypi.python.org/pypi/tornado>`_, :doc:`release notes <releases>`)
* `源代码 (github) <https://github.com/tornadoweb/tornado>`_
* 邮件列表: `discussion <http://groups.google.com/group/python-tornado>`_ and `announcements <http://groups.google.com/group/python-tornado-announce>`_
* `Stack Overflow <http://stackoverflow.com/questions/tagged/tornado>`_
* `Wiki <https://github.com/tornadoweb/tornado/wiki/Links>`_

Hello, world
------------
这里有一个简单的Tornado "Hello, world" Web app的demo::

    import tornado.ioloop
    import tornado.web

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            self.write("Hello, world")

    def make_app():
        return tornado.web.Application([
            (r"/", MainHandler),
        ])

    if __name__ == "__main__":
        app = make_app()
        app.listen(8888)
        tornado.ioloop.IOLoop.current().start()

这个demo没有使用任何Tornado的异步功能；关于异步功能的例子可以看这个 `简单的聊天室
<https://github.com/tornadoweb/tornado/tree/stable/demos/chat>`_.

线程和WSGI
----------------

Tornado不同于很多其他Python Web框架。它不是基于 `WSGI <https://wsgi.readthedocs.io/en/latest/>`_, 
并且通常每个进程只运行一个线程. 有关Tornado异步编程方法的更多信息，请参阅 :doc:`guide`

虽然在 `tornado.wsgi` 模块中提供了对WSGI的一些支持，但它不是开发的重点，并且大多数应用程序应该编写为直接使用Tornado自己的接口（例如 `tornado.web` ）而不是使用WSGI。

通常，Tornado代码不是线程安全的。 在Tornado中唯一可以安全地从其他线程调用的方法是 `IOLoop.add_callback`
您还可以使用 `IOLoop.run_in_executor` 在另一个线程上异步运行阻塞函数，但请注意，传递给run_in_executor的函数应该避免引用任何Tornado对象。
 ``run_in_executor`` 是与阻塞代码交互的推荐方法。


Installation
------------

::

    pip install tornado

Tornado is listed in `PyPI <http://pypi.python.org/pypi/tornado>`_ and
can be installed with ``pip``. Note that the source distribution
includes demo applications that are not present when Tornado is
installed in this way, so you may wish to download a copy of the
source tarball or clone the `git repository
<https://github.com/tornadoweb/tornado>`_ as well.

**Prerequisites**: Tornado 5.x runs on Python 2.7, and 3.4+ (Tornado
6.0 will require Python 3.5+; Python 2 will no longer be supported).
The updates to the `ssl` module in Python 2.7.9 are required (in some
distributions, these updates may be available in older python
versions). In addition to the requirements which will be installed
automatically by ``pip`` or ``setup.py install``, the following
optional packages may be useful:

* `pycurl <http://pycurl.sourceforge.net>`_ is used by the optional
  ``tornado.curl_httpclient``.  Libcurl version 7.22 or higher is required.
* `Twisted <http://www.twistedmatrix.com>`_ may be used with the classes in
  `tornado.platform.twisted`.
* `pycares <https://pypi.python.org/pypi/pycares>`_ is an alternative
  non-blocking DNS resolver that can be used when threads are not
  appropriate.
* `monotonic <https://pypi.python.org/pypi/monotonic>`_ or `Monotime
  <https://pypi.python.org/pypi/Monotime>`_ add support for a
  monotonic clock, which improves reliability in environments where
  clock adjustments are frequent. No longer needed in Python 3.

**Platforms**: Tornado should run on any Unix-like platform, although
for the best performance and scalability only Linux (with ``epoll``)
and BSD (with ``kqueue``) are recommended for production deployment
(even though Mac OS X is derived from BSD and supports kqueue, its
networking performance is generally poor so it is recommended only for
development use).  Tornado will also run on Windows, although this
configuration is not officially supported and is recommended only for
development use. Without reworking Tornado IOLoop interface, it's not
possible to add a native Tornado Windows IOLoop implementation or
leverage Windows' IOCP support from frameworks like AsyncIO or Twisted.

Documentation
-------------

This documentation is also available in `PDF and Epub formats
<https://readthedocs.org/projects/tornado/downloads/>`_.

.. toctree::
   :titlesonly:

   guide
   webframework
   http
   networking
   coroutine
   integration
   utilities
   faq
   releases

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`

Discussion and support
----------------------

You can discuss Tornado on `the Tornado developer mailing list
<http://groups.google.com/group/python-tornado>`_, and report bugs on
the `GitHub issue tracker
<https://github.com/tornadoweb/tornado/issues>`_.  Links to additional
resources can be found on the `Tornado wiki
<https://github.com/tornadoweb/tornado/wiki/Links>`_.  New releases are
announced on the `announcements mailing list
<http://groups.google.com/group/python-tornado-announce>`_.

Tornado is available under
the `Apache License, Version 2.0
<http://www.apache.org/licenses/LICENSE-2.0.html>`_.

This web site and all documentation is licensed under `Creative
Commons 3.0 <http://creativecommons.org/licenses/by/3.0/>`_.
