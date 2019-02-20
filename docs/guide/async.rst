异步和非阻塞I/O
---------------------------------

实时web功能需要为每个用户提供一个长链接。在传统的同步web服务器中，
这意味着为每个用户提供一个线程，但这个成本是很昂贵的。

为了尽量减少并发链接带来的开销，Tornado使用单线程事件循环的方式。
这意味着所有的应用代码都需要是异步非阻塞的，因为在任意时间只有一
个操作在执行。

异步和非阻塞的关系是非常近且经常交换使用的，但他们并不是相同的意思

阻塞
~~~~~~~~

一个函数在等待某些执行的返回值的时候会被 **阻塞** 。一个函数被阻塞有很多原因：
网络I/O，硬盘I/O，互斥锁等。事实上， *每个* 函数在运行和使用CPU的时候或多或少
会被阻塞(举个极端的例子来说明为什么对待CPU阻塞要和对待一般阻塞一样的认真,
例如hash加密 `bcrypt <http://bcrypt.sourceforge.net/>`_, 需要消耗几百毫秒的CPU
时间, 而这已经超过一般的网络或者磁盘I/O消耗的时间了)。

一个函数可以在某些方面阻塞另外一些方面阻塞。在Tornado中，我们一般讨论的阻塞指的是网络环境下的I/O阻塞，尽管所有的阻塞都被尽量减少了。

异步
~~~~~~~~~~~~

**异步** 函数在会在完成之前返回，在应用中触发下一个动作之前通常会在后台执行一些工作(和正常的 **同步**  函数在返回前就执行完所有的事情不同)。这里列 举了几种风格的异步接口:

* 回调参数
* 返回一个占位符 (`.Future`, ``Promise``, ``Deferred``)
* 将等待执行的动作添加到异步队列(例如 celery)
* 注册回调函数 (例如 POSIX signals)

不论使用哪种类型的接口, *按照定义*  异步函数与它们的调用者都有着不同的交互方式;也没有什么对调用者透明的方式使得同步函数异步(类似 
`gevent<http://www.gevent.org>`_ 使用轻量级线程的系统性能虽然堪比异步系统,但它们并没有真正的让事情异步).

Tornado中的异步操作通常返回占位符对象（``Futures``）, 例外的是像 `.IOLoop` 这样的底层组件使用回调。
 ``Futures`` 通常用 ``await`` or ``yield`` 关键字转换成结果.

例子
~~~~~~~~

一个简单的同步函数:

.. testcode::

    from tornado.httpclient import HTTPClient

    def synchronous_fetch(url):
        http_client = HTTPClient()
        response = http_client.fetch(url)
        return response.body

.. testoutput::
   :hide:

把上面的例子的同步函数用原生协程重写:

.. testcode::

   from tornado.httpclient import AsyncHTTPClient

   async def asynchronous_fetch(url):
       http_client = AsyncHTTPClient()
       response = await http_client.fetch(url)
       return response.body

.. testoutput::
   :hide:

或者与旧版本的Python兼容, 使用 `tornado.gen` 模块:

..  testcode::

    from tornado.httpclient import AsyncHTTPClient
    from tornado import gen

    @gen.coroutine
    def async_fetch_gen(url):
        http_client = AsyncHTTPClient()
        response = yield http_client.fetch(url)
        raise gen.Return(response.body)

协程是有点神奇，但它们在内部是这样实现的

.. testcode::

    from tornado.concurrent import Future

    def async_fetch_manual(url):
        http_client = AsyncHTTPClient()
        my_future = Future()
        fetch_future = http_client.fetch(url)
        def on_fetch(f):
            my_future.set_result(f.result().body)
        fetch_future.add_done_callback(on_fetch)
        return my_future

.. testoutput::
   :hide:

请注意，协程在执行结束之前返回其`.Future`。 这是协程 *异步* 的原因。

你可以用协同程序做任何事情，你也可以通过传递回调对象来达到目的，但协程提供了重要的手段，让你以类似同步代码的方式来组织代码。 这对于错误处理尤为重要
，因为``try`` /``except``块通过协程可以像你期望的那样正常工作，而这很难通过回调实现。
协程将在指南的下一节深入讨论。

