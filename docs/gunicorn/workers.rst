.. _tutorial_workers:

Workers
=======

.. contents::
    :local:

spawn_worker
------------

.. code-block:: python
   :linenos:

    def spawn_worker(self):
        self.worker_age += 1
        worker = self.worker_class(self.worker_age, self.pid, self.LISTENERS,
                                   self.app, self.timeout / 2.0,
                                   self.cfg, self.log)
        self.cfg.pre_fork(self, worker)
        pid = os.fork()
        if pid != 0:
            self.WORKERS[pid] = worker
            return pid

        # Process Child
        worker_pid = os.getpid()
        try:
            util._setproctitle("worker [%s]" % self.proc_name)
            self.log.info("Booting worker with pid: %s", worker_pid)
            if self.cfg.reuseport:
                worker.sockets = create_sockets(self.cfg, self.log)
                listeners_str = ",".join([str(l) for l in worker.sockets])
                self.log.info("Listening at: %s (%s) using reuseport",
                              listeners_str,
                              worker_pid)
            self.cfg.post_fork(self, worker)
            worker.init_process()
            sys.exit(0)
        except SystemExit:
            raise
        except AppImportError as e:
            self.log.debug("Exception while loading the application: \n%s",
                    traceback.format_exc())
            print("%s" % e, file=sys.stderr)
            sys.stderr.flush()
            sys.exit(self.APP_LOAD_ERROR)
        except:
            self.log.exception("Exception in worker process:\n%s",
                    traceback.format_exc())
            if not worker.booted:
                sys.exit(self.WORKER_BOOT_ERROR)
            sys.exit(-1)
        finally:
            self.log.info("Worker exiting (pid: %s)", worker_pid)
            try:
                worker.tmp.close()
                self.cfg.worker_exit(self, worker)
            except:
                pass

首先, 实例化 ``worker_class``, 执行 ``pre_fork`` 这个钩子,  然后就是fork子进程, 在父进程中记录
worker进程的pid和worker实例, ``self.WORKERS[pid] = worker``.
在子进程中, 设置进程名记录logger等, 然后执行 ``post_fork`` 钩子, 最后这个子进程进入自己的处理函数,
``worker.init_process``.

worker
------

.. code-block:: python
   :linenos:

    class Worker(object):

        SIGNALS = [getattr(signal, "SIG%s" % x)
                for x in "ABRT HUP QUIT INT TERM USR1 USR2 WINCH CHLD".split()]

        PIPE = []

        def __init__(self, age, ppid, sockets, app, timeout, cfg, log):
            """\
            This is called pre-fork so it shouldn't do anything to the
            current process. If there's a need to make process wide
            changes you'll want to do that in ``self.init_process()``.
            """
            self.age = age
            self.ppid = ppid
            self.sockets = sockets
            self.app = app
            self.timeout = timeout
            self.cfg = cfg
            self.booted = False
            self.aborted = False
            self.reloader = None

            self.nr = 0
            jitter = randint(0, cfg.max_requests_jitter)
            self.max_requests = cfg.max_requests + jitter or MAXSIZE
            self.alive = True
            self.log = log
            self.tmp = WorkerTmp(cfg)

        def __str__(self):
            return "<Worker %s>" % self.pid

        @property
        def pid(self):
            return os.getpid()

        def notify(self):
            """\
            Your worker subclass must arrange to have this method called
            once every ``self.timeout`` seconds. If you fail in accomplishing
            this task, the master process will murder your workers.
            """
            self.tmp.notify()

        def run(self):
            """\
            This is the mainloop of a worker process. You should override
            this method in a subclass to provide the intended behaviour
            for your particular evil schemes.
            """
            raise NotImplementedError()

        def init_process(self):
            """\
            If you override this method in a subclass, the last statement
            in the function should be to call this method with
            super(MyWorkerClass, self).init_process() so that the ``run()``
            loop is initiated.
            """

            # start the reloader
            if self.cfg.reload:
                def changed(fname):
                    self.log.info("Worker reloading: %s modified", fname)
                    os.kill(self.pid, signal.SIGQUIT)
                self.reloader = Reloader(callback=changed).start()

            # set environment' variables
            if self.cfg.env:
                for k, v in self.cfg.env.items():
                    os.environ[k] = v

            util.set_owner_process(self.cfg.uid, self.cfg.gid)

            # Reseed the random number generator
            util.seed()

            # For waking ourselves up
            self.PIPE = os.pipe()
            for p in self.PIPE:
                util.set_non_blocking(p)
                util.close_on_exec(p)

            # Prevent fd inheritance
            [util.close_on_exec(s) for s in self.sockets]
            util.close_on_exec(self.tmp.fileno())

            self.log.close_on_exec()

            self.init_signals()

            self.wsgi = self.app.wsgi()

            self.cfg.post_worker_init(self)

            # Enter main run loop
            self.booted = True
            self.run()

        def init_signals(self):
            # reset signaling
            [signal.signal(s, signal.SIG_DFL) for s in self.SIGNALS]
            # init new signaling
            signal.signal(signal.SIGQUIT, self.handle_quit)
            signal.signal(signal.SIGTERM, self.handle_exit)
            signal.signal(signal.SIGINT, self.handle_quit)
            signal.signal(signal.SIGWINCH, self.handle_winch)
            signal.signal(signal.SIGUSR1, self.handle_usr1)
            signal.signal(signal.SIGABRT, self.handle_abort)

            # Don't let SIGTERM and SIGUSR1 disturb active requests
            # by interrupting system calls
            if hasattr(signal, 'siginterrupt'):  # python >= 2.6
                signal.siginterrupt(signal.SIGTERM, False)
                signal.siginterrupt(signal.SIGUSR1, False)

        def handle_usr1(self, sig, frame):
            self.log.reopen_files()

        def handle_exit(self, sig, frame):
            self.alive = False

        def handle_quit(self, sig, frame):
            self.alive = False
            # worker_int callback
            self.cfg.worker_int(self)
            time.sleep(0.1)
            sys.exit(0)

        def handle_abort(self, sig, frame):
            self.alive = False
            self.cfg.worker_abort(self)
            sys.exit(1)

        def handle_error(self, req, client, addr, exc):
            request_start = datetime.now()
            addr = addr or ('', -1)  # unix socket case
            if isinstance(exc, (InvalidRequestLine, InvalidRequestMethod,
                    InvalidHTTPVersion, InvalidHeader, InvalidHeaderName,
                    LimitRequestLine, LimitRequestHeaders,
                    InvalidProxyLine, ForbiddenProxyRequest)):

                status_int = 400
                reason = "Bad Request"

                if isinstance(exc, InvalidRequestLine):
                    mesg = "Invalid Request Line '%s'" % str(exc)
                elif isinstance(exc, InvalidRequestMethod):
                    mesg = "Invalid Method '%s'" % str(exc)
                elif isinstance(exc, InvalidHTTPVersion):
                    mesg = "Invalid HTTP Version '%s'" % str(exc)
                elif isinstance(exc, (InvalidHeaderName, InvalidHeader,)):
                    mesg = "%s" % str(exc)
                    if not req and hasattr(exc, "req"):
                        req = exc.req  # for access log
                elif isinstance(exc, LimitRequestLine):
                    mesg = "%s" % str(exc)
                elif isinstance(exc, LimitRequestHeaders):
                    mesg = "Error parsing headers: '%s'" % str(exc)
                elif isinstance(exc, InvalidProxyLine):
                    mesg = "'%s'" % str(exc)
                elif isinstance(exc, ForbiddenProxyRequest):
                    reason = "Forbidden"
                    mesg = "Request forbidden"
                    status_int = 403

                msg = "Invalid request from ip={ip}: {error}"
                self.log.debug(msg.format(ip=addr[0], error=str(exc)))
            else:
                self.log.exception("Error handling request")

                status_int = 500
                reason = "Internal Server Error"
                mesg = ""

            if req is not None:
                request_time = datetime.now() - request_start
                environ = default_environ(req, client, self.cfg)
                environ['REMOTE_ADDR'] = addr[0]
                environ['REMOTE_PORT'] = str(addr[1])
                resp = Response(req, client, self.cfg)
                resp.status = "%s %s" % (status_int, reason)
                resp.response_length = len(mesg)
                self.log.access(resp, req, environ, request_time)

            try:
                util.write_error(client, status_int, reason, mesg)
            except:
                self.log.debug("Failed to send error message.")

        def handle_winch(self, sig, fname):
            # Ignore SIGWINCH in worker. Fixes a crash on OpenBSD.
            return

这个类中, 主要是初始化了一些信号

.. code:: python

    def init_signals(self):
        # reset signaling
        [signal.signal(s, signal.SIG_DFL) for s in self.SIGNALS]
        # init new signaling
        signal.signal(signal.SIGQUIT, self.handle_quit)
        signal.signal(signal.SIGTERM, self.handle_exit)
        signal.signal(signal.SIGINT, self.handle_quit)
        signal.signal(signal.SIGWINCH, self.handle_winch)
        signal.signal(signal.SIGUSR1, self.handle_usr1)
        signal.signal(signal.SIGABRT, self.handle_abort)

        # Don't let SIGTERM and SIGUSR1 disturb active requests
        # by interrupting system calls
        if hasattr(signal, 'siginterrupt'):  # python >= 2.6
            signal.siginterrupt(signal.SIGTERM, False)
            signal.siginterrupt(signal.SIGUSR1, False)

主要注意的是, ``signal.SIGTERN`` 这个信号, 它的处理函数是 ``handle_exit``.

.. code:: python

    def handle_exit(self, sig, frame):
        self.alive = False

只是将 worker 的状态标志为 ``self.alive = False``. 这就会使得worker能够在退出前处理未完成的请求.
在 ``init_process`` 的最后, 调用 ``self.run()`` 函数运行worker进程. ``self.run()`` 具体的实现需要看
worker的类型 ``self.worker_class``. 支持的类型定义在 ``gunicorn.workers.__init__.SUPPORTED_WORKERS`` 里.
