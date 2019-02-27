前言
------------

`Tornado <http://www.tornadoweb.org>`_ 是一个Python的web框架和异步网络库，
最初是在 `FriendFeed
<https://en.wikipedia.org/wiki/FriendFeed>`_ 开发的.   因为使用非阻塞网络I/O，Tornado可以支撑上万的网络连接
使其成为 `长轮询 <http://en.wikipedia.org/wiki/Push_technology#Long_polling>`_, `WebSockets <http://en.wikipedia.org/wiki/WebSocket>`_, 和其他需要与每个用户建立长期连接的应用程序的理想选择。

Tornado 大致可以分为四个主要部分：

* Web框架(包括 `.RequestHandler` 子类，用于创建Web应用和各种功能类(and various supporting classes).
* HTTP (`.HTTPServer` 和
  `.AsyncHTTPClient` )的客户端和服务器端实现。
* 异步网络库包括 `.IOLoop`
  和 `.IOStream` 类, 它们是用于构建HTTP服务器端的组件， 并且还可以用于实现其他协议。
* 协程库 (`tornado.gen`) 它允许比链式回调更直接的方式编写异步代码。这类似于Python 3.5开始的
  原生协程(``async def``). 如果可用，建议使用原生协成替代 `tornado.gen` 模块。

Tornado Web框架和HTTP服务器一起提供 `WSGI <http://www.python.org/dev/peps/pep-3333/>`_ 的全套替代方案。
虽然可以在WSGI容器 (`.WSGIAdapter`) 中使用Torando Web框架, 或者使用Tornado HTTP服务器作为其他WSGI框架 (`.WSGIContainer`) 的容器, 
但是这些组合中的每一个都有局限性，如果要充分利用Tornado，您将需要一起使用Tornado的Web框架和HTTP服务器。
