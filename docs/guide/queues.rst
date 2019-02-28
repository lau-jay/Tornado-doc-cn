:class:`~tornado.queues.Queue` 示例 - 一个并发web爬虫
================================================================

.. currentmodule:: tornado.queues

Tornado的 `tornado.queues` 模块实现了异步生产者 / 消费者模式的协程，
类似于Python标准库的 `queue` 模块为线程实现的模式.

一个协程会使用yield 暂停 `Queue.get` 直到队列中有值的时候才会继续执行。
如果队列设置了最大长度, 协程会yield暂停 `Queue.put` 直到队列中有空间才会继续。

一个 `~Queue` 会监控未完成任务的计数, 直到未完成任务计数为零。
`~Queue.put` 增加未完成任务计数; `~Queue.task_done` 减少未完成任务计数。

这个网络爬虫的例子，队列开始的时候只包含base_url。当一个worker抓取一个页面 并解析
链接把链接加入队列中，然后调用 `~Queue.task_done` 减少计数一次。
最后，当一个worker抓到的页面URL都是之前已经抓取过的并且队列中没有任务了。于是worker
调用 `~Queue.task_done` 把计数减到0。主协程 `~Queue.join` 等待挂起的执行完成。

.. literalinclude:: ../../tornado-stable/demos/webspider/webspider.py
