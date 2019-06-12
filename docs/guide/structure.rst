.. currentmodule:: tornado.web

.. testsetup::

   import tornado.web

Tornado Web应用的结构
======================================

通常一个Tornado web应用包括一个或者多个 `.RequestHandler` 子类,
一个可以将收到的请求路由到对应handler的 `.Application`  对象,
和 一个启动服务的 ``main()`` 函数。

一个最小的 "hello world" 例子就像下面这样:

.. testcode::

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

.. testoutput::
   :hide:

``Application`` 对象
~~~~~~~~~~~~~~~~~~~~~~~~~~

`.Application`  对象是负责全局配置的, 包括映射请求转发给处理程序的路由表。

路由表是 `.URLSpec` 对象(或元组)的列表, 其中每个都包含(至少)一个正则 表达式和一个处理类。
关于解析顺序; 第一个匹配的规则会被使用。如果正则表达 式包含捕获组，这些组会被作为 *路径参数* 传递给处理函数的HTTP方法。
如果一个字典作为 `.URLSpec` 的第三个参数被传递, 它会作为 *初始参数* 传递给 `.RequestHandler.initialize`。
最后 URLSpec 可能有一个名字，这将允许它被 `.RequestHandler.reverse_url` 使用。

例如, 在这个片段中根URL ``/`` 映射到了 ``MainHandler``，
像 ``/story/`` 后跟着一个数字这种形式的URL被映射到了 ``StoryHandler``。
这个数字被传递(作为字符串)给 ``StoryHandler.get``。

::

    class MainHandler(RequestHandler):
        def get(self):
            self.write('<a href="%s">link to story 1</a>' %
                       self.reverse_url("story", "1"))

    class StoryHandler(RequestHandler):
        def initialize(self, db):
            self.db = db

        def get(self, story_id):
            self.write("this is story %s" % story_id)

    app = Application([
        url(r"/", MainHandler),
        url(r"/story/([0-9]+)", StoryHandler, dict(db=db), name="story")
        ])


`.Application` 构造函数有很多关键字参数可以用于自定义应用程序的行为和使用某些特性(或者功能);
 完整列表请查看 `.Application.settings`。

``RequestHandler`` 子类
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Tornado web 应用程序的大部分工作是在 RequestHandler 子类下完成的。
处理子类的主入口点是一个命名为处理HTTP方法的函数: get(), post(), 等等。
每个处理程序可以定义一个或者多个这种方法来处理不同的HTTP动作。
如上所述, 这些方法将被匹配路由规则的捕获组对应的参数调用。

在处理程序中, 调用方法如 `.RequestHandler.render` 或者 `.RequestHandler.write` 产生一个响应。 ``render()`` 通过名字加载一个 `.Template` 并使用给定的参数渲染它。
``write()``  被用于非模板基础的输出;它接受字符串, 字节, 和字典(字典会被编码成JSON)。

在  `.RequestHandler` 中的很多方法的设计是为了在子类中复写和在整个应用中使用。
常用的方法是定义一个 ``BaseHandler`` 类,
复写一些方法例如 `~.RequestHandler.write_error` 和  `~.RequestHandler.get_current_user` 然
后子类继承使用你自己的 ``BaseHandler`` 而不是 `.RequestHandler` 在你所有具体的处理程序中.

处理请求
~~~~~~~~~~~~~~~~~~~~~~

处理器可以使用 ``self.request`` 访问当前请求对象。通过
`~tornado.httputil.HTTPServerRequest` 类的定义查看完整的属性列表。

通过form表单传递的请求数据会被解析并且能过通过 `~.RequestHandler.get_query_argument`
和  `~.RequestHandler.get_body_argument` 得到。

.. testcode::

    class MyFormHandler(tornado.web.RequestHandler):
        def get(self):
            self.write('<html><body><form action="/myform" method="POST">'
                       '<input type="text" name="message">'
                       '<input type="submit" value="Submit">'
                       '</form></body></html>')

        def post(self):
            self.set_header("Content-Type", "text/plain")
            self.write("You wrote " + self.get_body_argument("message"))

.. testoutput::
   :hide:

由form表单编码是不能确定一个标签的值是单一的还是一多个组成的列表  `.RequestHandler` 有明确的方式来允许应用程序表明它期望接收的值类型。对于列表，使用 `~.RequestHandler.get_query_arguments`
和 `~.RequestHandler.get_body_arguments` 而非获取单个值的方法。

通过表单上传的文件， 可以从 ``self.request.files`` 获取相关信息，它遍历文件名字(HTML标签 ``<input type="file">``) 到一个列表
。 每个文件都是一个字典形式 ``{"filename":..., "content_type":..., "body":...}`` 。
 ``files`` 对象如果文件上传是通过一个表单那么它是当前唯一的
(比如 ``multipart/form-data`` Content-Type); 反之可以使用 ``self.request.body`` 获取文件对象。
默认上传的文件是完全缓存在内存中的，如果你需要处理的文件比较大那么可以看看 `.stream_request_body` 类装饰器。

文件上传的例子： `file_receiver.py <https://github.com/tornadoweb/tornado/tree/master/demos/file_upload/>`_
展示了两种方式上传文件。

由于HTML表单编码格式的怪异 (比如 在单数和复数参数的含糊不清),
Tornado 不会试图统一表单参数和其他输入类型的参数。特别是, 我们不解析JSON请求体。
应用程序希望使用JSON代替表单编码可以复写  `~.RequestHandler.prepare`  来解析它们的请求::

    def prepare(self):
        if self.request.headers["Content-Type"].startswith("application/json"):
            self.json_args = json.loads(self.request.body)
        else:
            self.json_args = None


覆写RequestHandler方法
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

除了 ``get()``/``post()`` 之外，`.RequestHandler` 中的某些其他方法被设计为在必要时被子类覆盖。在每个请求中，都会发生以下调用时序:

1. 每个请求都会创建一个新新的 `.RequestHandler` 对象。
2. 使用Application配置中的初始化参数调用 `~.RequestHandler.initialize()`， ``initialize``
   通常应该只保存传递给成员变量的参数;它可能不会产生任何输出或调用
   `~.RequestHandler.send_error` 等方法。
3. `~.RequestHandler.prepare()` 被调用。这在所有处理程序子类共享的基类中最有用，因为无论使用哪种HTTP方法，都会调用 ``prepare`` 。 
    ``prepare`` 可以产生输出; 如果它调用 `~.RequestHandler.finish`（或 ``redirect`` 等），
    处理就在这里停止。

4. ``get()``，``post()``，``put()`` 等等， 其中一个方法会被执行。如果URL正则表达式包含捕获组，它们作为参数传递给方法。
5. 当请求完成后，调用 `~.RequestHandler.on_finish()` 方法。 对于大多数处理程序，这是在比如 ``get()`` 返回之后; 
   对于使用 `tornado.web.asynchronous` 装饰器的处理程序，它是在调用 `~.RequestHandler.finish()` 之后。

所有方法都被设计为可以覆写，更多的描述同样在 `.RequestHandler` 文档。 一些最常见的
被覆盖的方法包括:

- `~.RequestHandler.write_error` - 输出HTML以便在错误页上使用。
- `~.RequestHandler.on_connection_close` - 当客户端的链接断开时调用，应用程序可能会选择检测此情况并停止进一步处理。 请注意，无法保证关闭可以及时检测到链接。
- `~.RequestHandler.get_current_user` - 参见 :ref:`user-authentication`
- `~.RequestHandler.get_user_locale` - 返回 `.Locale` 对象以便给当前用户使用。
- `~.RequestHandler.set_default_headers` - 可用于在响应上设置其他标头（例如自定义 ``Server`` 标头）

错误处理
~~~~~~~~~~~~~~

如果处理程序引发异常，Tornado将调用 `.RequestHandler.write_error` 生成错误页面。
`tornado.web.HTTPError` 可用于生成指定的状态码;所有其他例外都返回500状态。

默认错误页面包括调试模式下的堆栈跟踪和错误的一行描述（例如“500：内部服务器错误”）。
要生成自定义错误页面，请覆写 `RequestHandler.write_error`
（可能在你的处理程序中所有人共享的基类中）。该方法可以正常产生输出诸如 `~RequestHandler.write` 和 `~RequestHandler.render` 之类的方法。
如果错误是由异常引起的，那么 ``exc_info`` 三元组将会出现， 作为关键字参数传递（
请注意，此异常不是保证是 `sys.exc_info` 中的当前异常，所以 ``write_error`` 必须使用例如 `traceback.format_exception` 而不是 `traceback.format_exc`）。

也可以从常规处理程序生成错误页面 而不是通过 ``write_error`` 调用 `~.RequestHandler.set_status`，写一个回复，然后返回。
可能会引发特殊的异常 `tornado.web.Finish` 来终止处理程序而不会在简单返回不方便的情况下调用 ``write_error``。

对于404错误，请使用应用程序设置 `<.Application.settings>` 中的 ``default_handler_class``。
这个处理程序应该覆写 `~.RequestHandler.prepare` 而不是像 ``get()`` 这样的更具体的方法，
所以它适用于任何HTTP方法。 它应该产生如上所述的错误页面：通过引发 ``HTTPError（404)`` 并覆写 ``write_error``，
或者调用 ``self.set_status（404）`` 并直接在producing the response directly in ``prepare()``。

重定向
~~~~~~~~~~~

在Tornado中有两个主要的方法重定向请求:
`.RequestHandler.redirect` 和 `.RedirectHandler`.

你可以在 `.RequestHandler` 对象中使用 ``self.redirect()`` 重定向用户到别处。
还有一个可选参数 ``permanent``，你可以用它来表示永久重定向。 ``permanent`` 的默认值为 ``False`` ,
这会生成 ``302 Found`` HTTP响应码适用于 ``POST`` 请求成功后重定向用户等。
如果 ``permanent`` 是 ``True`` , 将会返回 ``301 Moved
Permanently`` HTTP响应码，这是推荐的通常用于例如重定向到SEO友好的页面的
方式。

`.RedirectHandler` 让你直接在你的 `.Application` 配置中配置重定向
路由表。 例如，配置一个单个静态重定向::

    app = tornado.web.Application([
        url(r"/app", tornado.web.RedirectHandler,
            dict(url="http://itunes.apple.com/my-app-id")),
        ])

`.RedirectHandler` 也支持正则表达式替换。以下规则将重定向 ``/ pictures /`` 开头的所有请求
改为前缀 ``/ photos /`` ::

    app = tornado.web.Application([
        url(r"/photos/(.*)", MyPhotoHandler),
        url(r"/pictures/(.*)", tornado.web.RedirectHandler,
            dict(url=r"/photos/{0}")),
        ])

与 `.RequestHandler.redirect` 不同，`.RedirectHandler` 默认使用永久性重定向。
 这是因为在运行时路由表不会改变，被认为是永久性的，而在处理程序中找到的重定向可能是更改的其他逻辑的结果。
要使用 `.RedirectHandler` 发送临时重定向，请添加 ``permanent = False`` 到 `.RedirectHandler` 初始化参数。

异步处理器
~~~~~~~~~~~~~~~~~~~~~

某些处理程序方法（包括 ``prepare()`` 和HTTP动词方法 ``get()`` / ``post()``等）可以被重写为协程使处理程序异步。

Tornado还支持使用 `tornado.web.asynchronous` 装饰器的基于回调的异步处理程序，但这种方式已弃用，将在Tornado 6.0中删除。
新应用程序应该使用协程。

例如，这是一个使用协程的简单处理程序:

.. testcode::

    class MainHandler(tornado.web.RequestHandler):
        async def get(self):
            http = tornado.httpclient.AsyncHTTPClient()
            response = await http.fetch("http://friendfeed-api.com/v2/feed/bret")
            json = tornado.escape.json_decode(response.body)
            self.write("Fetched " + str(len(json["entries"])) + " entries "
                       "from the FriendFeed API")

.. testoutput::
   :hide:

有关更高级的异步示例，请查看 `chat` 示例应用 `<https://github.com/tornadoweb/tornado/tree/stable/demos/chat>` _, 它使用 `长轮询` 实现了一个AJAX聊天室
 `<http://en.wikipedia.org/wiki/Push_technology#Long_polling>` _ 。长轮询的用户可能希望覆写 
 ``on_connection_close（）`` 以在客户端关闭连接后进行清理（但请注意该方法的文档字符串以正确使用) 。
