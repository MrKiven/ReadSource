.. _tutorial_arbiter:

Main master loop
=================

.. contents::
    :local:

``arbiter.Arbiter`` 这个类就是一个 `main master loop`.

run
---

首先看 ``run`` 这个方法:

.. code-block:: python
   :linenos:

    def run(self):
        "Main master loop."
        self.start()
        util._setproctitle("master [%s]" % self.proc_name)

        try:
            self.manage_workers()

            while True:
                self.maybe_promote_master()

                sig = self.SIG_QUEUE.pop(0) if len(self.SIG_QUEUE) else None
                if sig is None:
                    self.sleep()
                    self.murder_workers()
                    self.manage_workers()
                    continue

                if sig not in self.SIG_NAMES:
                    self.log.info("Ignoring unknown signal: %s", sig)
                    continue

                signame = self.SIG_NAMES.get(sig)
                handler = getattr(self, "handle_%s" % signame, None)
                if not handler:
                    self.log.error("Unhandled signal: %s", signame)
                    continue
                self.log.info("Handling signal: %s", signame)
                handler()
                self.wakeup()
        except StopIteration:
            self.halt()
        except KeyboardInterrupt:
            self.halt()
        except HaltServer as inst:
            self.halt(reason=inst.reason, exit_status=inst.exit_status)
        except SystemExit:
            raise
        except Exception:
            self.log.info("Unhandled exception in main loop",
                          exc_info=True)
            self.stop(False)
            if self.pidfile is not None:
                self.pidfile.unlink()
            sys.exit(-1)

这个方法就是一个死循环, 不断的检测和管理 ``worker``. 所以 ``master`` 其实没干什么事.
只是用来管理 ``worker`` 的. 而 ``worker`` 才是真正干活的. 这个后面会详细说到.

self.start
^^^^^^^^^^

现在我们来看 ``run`` 函数的第三行: ``self.start()``, 这个 ``start`` 就是用来
``Initialize the arbiter. Start listening and set pidfile if needed.``


.. code-block:: python
   :linenos:

    def start(self):
        """\
        Initialize the arbiter. Start listening and set pidfile if needed.
        """
        self.log.info("Starting gunicorn %s", __version__)

        if 'GUNICORN_PID' in os.environ:
            self.master_pid = int(os.environ.get('GUNICORN_PID'))
            self.proc_name = self.proc_name + ".2"
            self.master_name = "Master.2"

        self.pid = os.getpid()
        if self.cfg.pidfile is not None:
            pidname = self.cfg.pidfile
            if self.master_pid != 0:
                pidname += ".2"
            self.pidfile = Pidfile(pidname)
            self.pidfile.create(self.pid)
        self.cfg.on_starting(self)

        self.init_signals()
        if not self.LISTENERS:
            self.LISTENERS = create_sockets(self.cfg, self.log)

        listeners_str = ",".join([str(l) for l in self.LISTENERS])
        self.log.debug("Arbiter booted")
        self.log.info("Listening at: %s (%s)", listeners_str, self.pid)
        self.log.info("Using worker: %s", self.cfg.worker_class_str)

        # check worker class requirements
        if hasattr(self.worker_class, "check_config"):
            self.worker_class.check_config(self.cfg, self.log)

        self.cfg.when_ready(self)

这个函数主要是看最后一段代码, 这个时候是 ``master`` 初始化后, 执行 ``when_ready`` 这个钩子.
这个钩子, 我们可以做一些事情, 给个例子:

.. code:: python

    # this is your application file

    from gunicron.app.base import Application

    def when_ready(server):
        """when gunicorn master start, do something yourself"""


    class MyApp(Application):
        def install_hooks(self):
            self.cfg.set('when_ready', when_ready)

self.manage_workers()
^^^^^^^^^^^^^^^^^^^^^^

.. code:: python

    def manage_workers(self):
        """\
        Maintain the number of workers by spawning or killing
        as required.
        """
        if len(self.WORKERS.keys()) < self.num_workers:
            self.spawn_workers()

        workers = self.WORKERS.items()
        workers = sorted(workers, key=lambda w: w[1].age)
        while len(workers) > self.num_workers:
            (pid, _) = workers.pop(0)
            self.kill_worker(pid, signal.SIGTERM)

        active_worker_count = len(workers)
        if self._last_logged_active_worker_count != active_worker_count:
            self._last_logged_active_worker_count = active_worker_count
            self.log.debug("{0} workers".format(active_worker_count),
                           extra={"metric": "gunicorn.workers",
                                  "value": active_worker_count,
                                  "mtype": "gauge"})

这个函数就是用来保证 ``worker`` 数量, 主要是通过 ``spawn_workers()`` 和 ``kill_worker``
这两个函数.
