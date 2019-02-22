协程
==========

.. testsetup::

   from tornado import gen

**Coroutines** are the recommended way to write asynchronous code in
Tornado. Coroutines use the Python ``await`` or ``yield`` keyword to
suspend and resume execution instead of a chain of callbacks
(cooperative lightweight threads as seen in frameworks like `gevent
<http://www.gevent.org>`_ are sometimes called coroutines as well, but
in Tornado all coroutines use explicit context switches and are called
as asynchronous functions).
Tornado中推荐通过 **协程** 的方式写异步代码.  协程使用了Python ``yield`` 关键字来挂起和恢复执行程序而不是链式回调(像在 `gevent<http://www.gevent.org>`_ 中出现的轻量级线程有时也被称为协程, 但Tornado中所有协程都采用显式的上下文切换并作为异步函数被调用).



Coroutines are almost as simple as synchronous code, but without the
expense of a thread.  They also `make concurrency easier
<https://glyph.twistedmatrix.com/2014/02/unyielding.html>`_ to reason
about by reducing the number of places where a context switch can
happen.
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

两种表格之间的转换通常很简单::

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

Other differences between the two forms of coroutine are:

- Native coroutines are generally faster.
- Native coroutines can use ``async for`` and ``async with``
  statements which make some patterns much simpler.
- Native coroutines do not run at all unless you ``await`` or
  ``yield`` them. Decorated coroutines can start running "in the
  background" as soon as they are called. Note that for both kinds of
  coroutines it is important to use ``await`` or ``yield`` so that
  any exceptions have somewhere to go.
- Decorated coroutines have additional integration with the
  `concurrent.futures` package, allowing the result of
  ``executor.submit`` to be yielded directly. For native coroutines,
  use `.IOLoop.run_in_executor` instead.
- Decorated coroutines support some shorthand for waiting on multiple
  objects by yielding a list or dict. Use `tornado.gen.multi` to do
  this in native coroutines.
- Decorated coroutines can support integration with other packages
  including Twisted via a registry of conversion functions.
  To access this functionality in native coroutines, use
  `tornado.gen.convert_yielded`.
- Decorated coroutines always return a `.Future` object. Native
  coroutines return an *awaitable* object that is not a `.Future`. In
  Tornado the two are mostly interchangeable.

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

In nearly all cases, any function that calls a coroutine must be a
coroutine itself, and use the ``await`` or ``yield`` keyword in the
call. When you are overriding a method defined in a superclass,
consult the documentation to see if coroutines are allowed (the
documentation should say that the method "may be a coroutine" or "may
return a `.Future`")::

    async def good_call():
        # await will unwrap the object returned by divide() and raise
        # the exception.
        await divide(1, 0)

Sometimes you may want to "fire and forget" a coroutine without waiting
for its result. In this case it is recommended to use `.IOLoop.spawn_callback`,
which makes the `.IOLoop` responsible for the call. If it fails,
the `.IOLoop` will log a stack trace::

    # The IOLoop will catch the exception and print a stack trace in
    # the logs. Note that this doesn't look like a normal call, since
    # we pass the function object to be called by the IOLoop.
    IOLoop.current().spawn_callback(divide, 1, 0)

Using `.IOLoop.spawn_callback` in this way is *recommended* for
functions using ``@gen.coroutine``, but it is *required* for functions
using ``async def`` (otherwise the coroutine runner will not start).

Finally, at the top level of a program, *if the IOLoop is not yet
running,* you can start the `.IOLoop`, run the coroutine, and then
stop the `.IOLoop` with the `.IOLoop.run_sync` method. This is often
used to start the ``main`` function of a batch-oriented program::

    # run_sync() doesn't take arguments, so we must wrap the
    # call in a lambda.
    IOLoop.current().run_sync(lambda: divide(1, 0))

Coroutine patterns
~~~~~~~~~~~~~~~~~~

Calling blocking functions
^^^^^^^^^^^^^^^^^^^^^^^^^^

The simplest way to call a blocking function from a coroutine is to
use `.IOLoop.run_in_executor`, which returns
``Futures`` that are compatible with coroutines::

    async def call_blocking():
        await IOLoop.current().run_in_executor(None, blocking_func, args)

Parallelism
^^^^^^^^^^^

The `.multi` function accepts lists and dicts whose values are
``Futures``, and waits for all of those ``Futures`` in parallel:

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

In decorated coroutines, it is possible to ``yield`` the list or dict directly::

    @gen.coroutine
    def parallel_fetch_decorated(url1, url2):
        resp1, resp2 = yield [http_client.fetch(url1),
                              http_client.fetch(url2)]

Interleaving
^^^^^^^^^^^^

Sometimes it is useful to save a `.Future` instead of yielding it
immediately, so you can start another operation before waiting.

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

This is a little easier to do with decorated coroutines, because they
start immediately when called:

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

Looping
^^^^^^^

In native coroutines, ``async for`` can be used. In older versions of
Python, looping is tricky with coroutines since there is no way to
``yield`` on every iteration of a ``for`` or ``while`` loop and
capture the result of the yield. Instead, you'll need to separate the
loop condition from accessing the results, as in this example from
`Motor <https://motor.readthedocs.io/en/stable/>`_::

    import motor
    db = motor.MotorClient().test

    @gen.coroutine
    def loop_example(collection):
        cursor = db.collection.find()
        while (yield cursor.fetch_next):
            doc = cursor.next_object()

Running in the background
^^^^^^^^^^^^^^^^^^^^^^^^^

`.PeriodicCallback` is not normally used with coroutines. Instead, a
coroutine can contain a ``while True:`` loop and use
`tornado.gen.sleep`::

    async def minute_loop():
        while True:
            await do_something()
            await gen.sleep(60)

    # Coroutines that loop forever are generally started with
    # spawn_callback().
    IOLoop.current().spawn_callback(minute_loop)

Sometimes a more complicated loop may be desirable. For example, the
previous loop runs every ``60+N`` seconds, where ``N`` is the running
time of ``do_something()``. To run exactly every 60 seconds, use the
interleaving pattern from above::

    async def minute_loop2():
        while True:
            nxt = gen.sleep(60)   # Start the clock.
            await do_something()  # Run while the clock is ticking.
            await nxt             # Wait for the timer to run out.
