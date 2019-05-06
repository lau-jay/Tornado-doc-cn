协程
==========

.. testsetup::

   from tornado import gen

Tornado中推荐通过 **协程** 的方式写异步代码.  协程使用了Python  ``await`` or ``yield`` 关键字来挂起和恢复执行程序而不是链式回调(像在  `gevent<http://www.gevent.org>`_  中出现的轻量级线程有时也被称为协程, 但Tornado中所有协程都采用显式的上下文切换并作为异步函数被调用).

使用协程方式来写异步代码几乎跟写同步代码一样简单，并不需额外的线程开销。 还通过减少上下文切换来 `使并发编程更简单
<https://glyph.twistedmatrix.com/2014/02/unyielding.html>`_ 。

例子::

    async def fetch_coroutine(url):
        http_client = AsyncHTTPClient()
        response = await http_client.fetch(url)
        return response.body

.. _native_coroutines:

原生协 vs 装饰器协程
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Python 3.5 引入了 ``async`` 和 ``await`` 关键字(使用这些关键字的函数也被称为”原生协程”)。
为了兼容旧版本的Python，你可以使用"装饰器"或"基于yield"的协程在函数定义的时候使用 ``@gen.coroutine`` 装饰器, 并且
用 yield代替await.

尽可能使用原生协程。 只在为里兼容旧版本的Python时使用装饰协程。
Tornado的文档里通常都使用原生协程。

两种形式之间的转换通常很简单::

    # 装饰器形式:                    # 原生:

    # Normal function declaration
    # with decorator                # "async def" keywords
    @gen.coroutine
    def a():                        async def a():
        # "yield" all async funcs       # "await" all async funcs
        b = yield c()                   b = await c()
        # "return" and "yield"
        # cannot be mixed in
        # Python 2, so raise a
        # special exception.            # Return normally
        raise gen.Return(b)             return b

以下是两者之间协程的其他不同之处:

- 原生协程通常更快
- 原生协程能使用 ``async for`` 和 ``async with``
  语法这使得一些模式更简单。
- 除非你 ``await`` 或 ``yield`` 它们，否则原生协程都不会运行。
  装饰器协程一旦被调用就可以“在后台”开始运行。 
  请注意，对于这两种协同程序，使用 ``await`` 或 ``yield`` 非常重要，
  这样任何异常都可以使用。
- 装饰器协程与 `concurrent.futures` 包有额外的集成，允许直接生成 ``executor.submit`` 的结果。 对于本地协同程序，
   使用 `.IOLoop.run_in_executor` 代替。
- 装饰器协程通过yield列表或字典来简化支持等待多个对象。 使用`tornado.gen.multi`在原生协程中执行此操作。
- 装饰器协程可以支持与其他包的集成 包括通过转换函数注册表的Twisted。要在本机协同程序中访问此功能，请使用 `tornado.gen.convert_yielded`。
- 装饰器协程始终返回 `.Future` 对象。原生协程返回一个不是 `.Future` 的 *awaitable* 对象。在Troando中，两者大多可以互换。

它是如何工作的
~~~~~~~~~~~~

这一段解释基于装饰器的协程。原生协程在概念上是基本类似的，但有一点
复杂因为与Python运行时的额外集成有关。

一个函数包含了关键字 ``yield`` , 那么这个函数是一个生成器。所有的生成器都是异步的；
当调用生成器的时候会返回一个生成器对象而不是该生成器执行的结果。使用 ``@gen.coroutine`` 装饰器
装饰的生成器, 如此产生的协程的调用者能通过 ``yield`` 表达式和协程进行通信，并且调用者可以获得一个 `.Future` 与协程进行交互。

下面是一个协程装饰器内部循环的简单版本::

    # Simplified inner loop of tornado.gen.Runner
    def run(self):
        # send(x) makes the current yield return x.
        # It returns when the next yield is reached
        future = self.gen.send(self.next)
        def callback(f):
            self.next = f.result()
            self.run()
        future.add_done_callback(callback)

装饰器从生成器接收一个 `.Future` 对象，等待(非阻塞)这个 `.Future` 对象执行完成，
然后解开('unwrap')这个 `.Future` 对象，并把结果作为 `yield` 的结果传回给生成器器。
大多数异步代码从来不会接触到 `.Future` 对象, 除非立即将由异步函数返回的`.Future`传递给 `yield` 表达式

如何调用协程
~~~~~~~~~~~~~~~~~~~~~~~

协程一般不会抛异常: 任何异常都会被awaitable捕获直到它被返回。这意味着正确地调用协程很重要，
否则可能会无法得到错误信息::

    async def divide(x, y):
        return x / y

    def bad_call():
        # This should raise a ZeroDivisionError, but it won't because
        # the coroutine is called incorrectly.
        divide(1, 0)

在几乎所有情况下，调用协程的任何函数都必须是协程本身，并在调用中使用await或yield关键字。
当覆写父类中定义的方法时，请查阅文档以查看是否允许支持协程（文档应该说方法“可能是协程”或“可能返回 `.Future`”）::

    async def good_call():
        # await will unwrap the object returned by divide() and raise
        # the exception.
        await divide(1, 0)

有时你可能想要“fire and forget”一个协程而不等待它的结果。
在这种情况下，建议使用 `.IOLoop.spawn_callback`，
这使得 `.IOLoop` 负责调用。如果失败，`.IOLoop` 将记录堆栈::
    # The IOLoop will catch the exception and print a stack trace in
    # the logs. Note that this doesn't look like a normal call, since
    # we pass the function object to be called by the IOLoop.
    IOLoop.current().spawn_callback(divide, 1, 0)

使用 `.IOLoop.spawn_callback` 这种方式in this way is *recommended* for
functions using ``@gen.coroutine``, but it is *required* for functions
using ``async def`` (otherwise the coroutine runner will not start).
对于使用@ gen.coroutine的函数，建议以这种方式使用IOLoop.spawn_callback，
但是使用async def的函数需要它（否则协程运行器将无法启动）。

最后，在程序的顶层，* 如果IOLoop尚未运行 * ，您可以启动 `.IOLoop`，
使用 `.IOLoop.run_sync` 方法运行协程，然后停止IOLoop并得到协程的结果。 
这通常用于启动入口函数 ``main`` 面向批处理的程序::
    # run_sync() doesn't take arguments, so we must wrap the
    # call in a lambda.
    IOLoop.current().run_sync(lambda: divide(1, 0))

协程模式
~~~~~~~~~~~~~~~~~~

调用阻塞函数
^^^^^^^^^^^^^^^^^^^^^^^^^^

在协程中调用阻塞函数的最简单方法是使用  `.IOLoop.run_in_executor`, 返回一个与协程兼容的 ``Futures`` ::
    async def call_blocking():
        await IOLoop.current().run_in_executor(None, blocking_func, args)

并行
^^^^^^^^^^^

该 `.multi` 函数接受值为 ``Futures`` 的列表和字典，并等待所有这些  ``Futures``  并行执行:

.. testcode::

    from tornado.gen import multi

    async def parallel_fetch(url1, url2):
        resp1, resp2 = await multi([http_client.fetch(url1),
                                    http_client.fetch(url2)])

    async def parallel_fetch_many(urls):
        responses = await multi ([http_client.fetch(url) for url in urls])
        # responses is a list of HTTPResponses in the same order

    async def parallel_fetch_dict(urls):
        responses = await multi({url: http_client.fetch(url)
                                 for url in urls})
        # responses is a dict {url: HTTPResponse}

.. testoutput::
   :hide:

在装饰器协程中，可以直接 ``yield``  列表或字典::

    @gen.coroutine
    def parallel_fetch_decorated(url1, url2):
        resp1, resp2 = yield [http_client.fetch(url1),
                              http_client.fetch(url2)]

Interleaving
^^^^^^^^^^^^
有时保存 `.Future` 而不是立即产生值是有用的，因为你可以等待之前启动另外一个操作。

.. testcode::

    from tornado.gen import convert_yielded

    async def get(self):
        # convert_yielded() starts the native coroutine in the background.
        # This is equivalent to asyncio.ensure_future() (both work in Tornado).
        fetch_future = convert_yielded(self.fetch_next_chunk())
        while True:
            chunk = yield fetch_future
            if chunk is None: break
            self.write(chunk)
            fetch_future = convert_yielded(self.fetch_next_chunk())
            yield self.flush()

.. testoutput::
   :hide:

这对于装饰器协程来说更容易一些，因为他们在被调用时立即启动:

.. testcode::

    @gen.coroutine
    def get(self):
        fetch_future = self.fetch_next_chunk()
        while True:
            chunk = yield fetch_future
            if chunk is None: break
            self.write(chunk)
            fetch_future = self.fetch_next_chunk()
            yield self.flush()

.. testoutput::
   :hide:

循环
^^^^^^^

在原生协程里，能够使用``async for`` 。在旧版本的Python中，使用协程循环很棘手，因为
无法在for循环或while循环的每次迭代中捕获yield的结果。事实上，你需要将循环条件和访问结果分开，比如
`Motor <https://motor.readthedocs.io/en/stable/>`_ 中的例子::

    import motor
    db = motor.MotorClient().test

    @gen.coroutine
    def loop_example(collection):
        cursor = db.collection.find()
        while (yield cursor.fetch_next):
            doc = cursor.next_object()


后台运行
^^^^^^^^^^^^^^^^^^^^^^^^^

`.PeriodicCallback` 通常不是用在协程里的。事实上，一个协程能包含一个 ``while True:`` 循环和使用
`tornado.gen.sleep`::

    async def minute_loop():
        while True:
            await do_something()
            await gen.sleep(60)

    # 死循环的协同程序通常使用
    # spawn_callback()执行
    IOLoop.current().spawn_callback(minute_loop)

有时可能需要更复杂的循环。例如，上一个循环每 ``60 + N`` 秒运行一次，
其中 ``N`` 是`` do_something()`` 的运行时间。要完全每60秒运行一次，使用上面的方式交错执行::

    async def minute_loop2():
        while True:
            nxt = gen.sleep(60)   # 开始计时
            await do_something()  # 在计时的同时切换执行do_something
            await nxt             # 等待计时结束
