.. _tutorial_gracefultimeout:

Graceful timeout
================

本章主要介绍 gunicorn 是如何 gracefully 退出的.

gunicorn 需要优雅退出必须是 master 接受到一个 `SIGTERM` 信号( `man kill` 查看相关信息).

看一下 master 的信号处理器（信号处理函数）:

.. code-block:: python
  :linenos:

    def handle_term(self):
        "SIGTERM handling"
        raise StopIteration

    def handle_int(self):
        "SIGINT handling"
        self.stop(False)
        raise StopIteration

    def handle_quit(self):
        "SIGQUIT handling"
        self.stop(False)
        raise StopIteration

    def stop(self, graceful=True):
        """\
        Stop workers

        :attr graceful: boolean, If True (the default) workers will be
        killed gracefully  (ie. trying to wait for the current connection)
        """
        self.LISTENERS = []
        sig = signal.SIGTERM
        if not graceful:
            sig = signal.SIGQUIT
        limit = time.time() + self.cfg.graceful_timeout
        # instruct the workers to exit
        self.kill_workers(sig)
        # wait until the graceful timeout
        while self.WORKERS and time.time() < limit:
            time.sleep(0.1)

        self.kill_workers(signal.SIGKILL)

上面只列出了几个关键函数，可以看到，master 在收到 `SIGTERM` 信号之后是直接 raise 一个 `StopIteration` ,
在 master 的主循环中是有 `catch` 这个异常的

.. code-block:: python
  :linenos:

    while True:
        try:
            # 省略一些代码
        except StopIteration:
            self.halt()
        except KeyboardInterrupt:
            self.halt()

        # 省略一些代码

所以 master 在收到 `SIGTERM` 信号之后调用 `self.halt()` 函数，看一看这个函数:

.. code-block:: python

    def halt(self, reason=None, exit_status=0):
        """ halt arbiter """
        self.stop()
        self.log.info("Shutting down: %s", self.master_name)
        if reason is not None:
            self.log.info("Reason: %s", reason)
        if self.pidfile is not None:
            self.pidfile.unlink()
        self.cfg.on_exit(self)
        sys.exit(exit_status)

跟收到其他信号一样，调用 `self.stop()` , 这个函数已经在上方代码里列出了。

1. 清空 `self.LISTENERS`, 这个属性里存放的就是建立的 `socket` 实例，一般来说这个列表里只有一个 `socket`

2. 设置默认发给 `worker` 的信号为 `SIGTERM`, 如果参数 `graceful` 非 `True` (默认)，则改为 `SIGQUIT` 信号.

3. 给所有 `worker` 发送第二步中的信号.

4. 如果有 `worker` 还没有退出则等待, 直到所有 `worker` 退出或者等待时间超过 `config` 中配置的 `graceful_timeout` (默认为30秒)

5. 给所有超过 `graceful_timeout` 还没退出的 `worker` 发送 `SIGKILL`

至此， `master` 是如何通知 `worker` 优雅退出的已经讲完，下一章将剖析 `worker` 收到 `master` 发过来的信号后是如何处理的。
