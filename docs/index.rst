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
您还可以使用 `IOLoop.run_in_executor` 在另一个线程上异步运行阻塞函数，但请注意，传递给run_in_executor的函数应该避免引用任何Tornado对象。 ``run_in_executor`` 是与阻塞代码交互的推荐方法。


安装Tornado
------------

::

    pip install tornado

在 `PyPI <http://pypi.python.org/pypi/tornado>`_ 中可以用 ``pip`` 安装.
请注意，源代码分发包括以这种方式安装Tornado时不存在demo应用程序，
因此您可能希望下载源代码的tar包或克隆 `git存储库
<https://github.com/tornadoweb/tornado>`_ .

**先决条件**: 

Tornado 5.x 运行需要的Python版本为Python 2.7, 或3.4+ (Tornado
6.0 可能需要Python 3.5+; Python 2 将不被支持). 需要对Python 2.7.9中的ssl模块进行更新（在某些发行版中，这些更新可能在较旧的python版本中可用）。 
除了将由 ``pip`` 或 ``setup.py install`` 自动安装的依赖之外，以下可选包可能很有用:

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

**系统平台**: Tornado 应该运行在任意Unix/Linux系统, 为了获得最佳性能和扩展性，建议只将
 Linux (有 ``epoll``)和 BSD (有 ``kqueue``) 作为生产环境的部署系统
(即使Mac OSX源自BSD并支持kqueue，但其网络性能通常很差，因此建议仅限于开发用途)。
Tornado当然也可以在windows上运行，但是这个方式不受官方支持，建议仅用于开发。
如果不重新设计Tornado IOLoop接口，那么将不可能添加原生的Tornado Windows IOLoop 实现或
利用 ``IOCP`` 提供像AsyncIO或Twisted那样的框架级支持。

文档
-------------

这份文档的英文版本同样提供 `PDF 和Epub格式的，中文暂时只提供在线。
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
