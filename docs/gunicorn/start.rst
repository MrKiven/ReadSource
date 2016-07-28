.. _start:

从哪里开始
----------

首先从gunicorn的 `setup.py` 开始吧.
里面的 `entry_points` 里这样:

.. code:: python

    entry_points="""
    [console_scripts]
    gunicorn=gunicorn.app.wsgiapp:run
    gunicorn_django=gunicorn.app.djangoapp:run
    gunicorn_paster=gunicorn.app.pasterapp:run

    [paste.server_runner]
    main=gunicorn.app.pasterapp:paste_server
    """

所以我们从最基础的 `wsgiapp` 看起.

WSGIApplication
^^^^^^^^^^^^^^^^

入口就在这里:

.. code:: python

    def run():
        """Here is docstring"""
        from gunicorn.app.wsgiapp import WSGIApplication
        WSGIApplication("%(prog)s [OPTIONS] [APP_MODULE]").run()

然后我们可以看到 `WSGIApplication` 这个class, 其实代码也不长:

.. code:: python

    class WSGIApplication(Application):
    def init(self, parser, opts, args):
        if opts.paste and opts.paste is not None:
            app_name = 'main'
            path = opts.paste
            if '#' in path:
                path, app_name = path.split('#')
            path = os.path.abspath(os.path.normpath(
                os.path.join(util.getcwd(), path)))

            if not os.path.exists(path):
                raise ConfigError("%r not found" % path)

            # paste application, load the config
            self.cfgurl = 'config:%s#%s' % (path, app_name)
            self.relpath = os.path.dirname(path)

            from .pasterapp import paste_config
            return paste_config(self.cfg, self.cfgurl, self.relpath)

        if len(args) < 1:
            parser.error("No application module specified.")

        self.cfg.set("default_proc_name", args[0])
        self.app_uri = args[0]

    def chdir(self):
        # chdir to the configured path before loading,
        # default is the current dir
        os.chdir(self.cfg.chdir)

        # add the path to sys.path
        sys.path.insert(0, self.cfg.chdir)

    def load_wsgiapp(self):
        self.chdir()

        # load the app
        return util.import_app(self.app_uri)

    def load_pasteapp(self):
        self.chdir()

        # load the paste app
        from .pasterapp import load_pasteapp
        return load_pasteapp(self.cfgurl, self.relpath, global_conf=self.cfg.paste_global_conf)

    def load(self):
        if self.cfg.paste is not None:
            return self.load_pasteapp()
        else:
            return self.load_wsgiapp()

可以看到这个 `WSGIApplication` 类继承自 `Application` .
所以, 我们需要进入到这个父类里一探究竟.

.. code:: python

    class Application(BaseApplication):

    def get_config_from_filename(self, filename):

        if not os.path.exists(filename):
            raise RuntimeError("%r doesn't exist" % filename)

        cfg = {
            "__builtins__": __builtins__,
            "__name__": "__config__",
            "__file__": filename,
            "__doc__": None,
            "__package__": None
        }
        try:
            execfile_(filename, cfg, cfg)
        except Exception:
            print("Failed to read config file: %s" % filename, file=sys.stderr)
            traceback.print_exc()
            sys.stderr.flush()
            sys.exit(1)

        return cfg

    def get_config_from_module_name(self, module_name):
        return util.import_module(module_name).__dict__

    def load_config_from_module_name_or_filename(self, location):
        """
        Loads the configuration file: the file is a python file, otherwise raise an RuntimeError
        Exception or stop the process if the configuration file contains a syntax error.
        """

        if location.startswith("python:"):
            module_name = location[len("python:"):]
            cfg = self.get_config_from_module_name(module_name)
        else:
            if location.startswith("file:"):
                filename = location[len("file:"):]
            else:
                filename = location
            cfg = self.get_config_from_filename(filename)

        for k, v in cfg.items():
            # Ignore unknown names
            if k not in self.cfg.settings:
                continue
            try:
                self.cfg.set(k.lower(), v)
            except:
                print("Invalid value for %s: %s\n" % (k, v), file=sys.stderr)
                sys.stderr.flush()
                raise

        return cfg

    def load_config_from_file(self, filename):
        return self.load_config_from_module_name_or_filename(location=filename)

    def load_config(self):
        # parse console args
        parser = self.cfg.parser()
        args = parser.parse_args()

        # optional settings from apps
        cfg = self.init(parser, args, args.args)

        # Load up the any app specific configuration
        if cfg and cfg is not None:
            for k, v in cfg.items():
                self.cfg.set(k.lower(), v)

        if args.config:
            self.load_config_from_file(args.config)
        else:
            default_config = get_default_config_file()
            if default_config is not None:
                self.load_config_from_file(default_config)

        # Lastly, update the configuration with any command line
        # settings.
        for k, v in args.__dict__.items():
            if v is None:
                continue
            if k == "args":
                continue
            self.cfg.set(k.lower(), v)

    def run(self):
        if self.cfg.check_config:
            try:
                self.load()
            except:
                msg = "\nError while loading the application:\n"
                print(msg, file=sys.stderr)
                traceback.print_exc()
                sys.stderr.flush()
                sys.exit(1)
            sys.exit(0)

        if self.cfg.spew:
            debug.spew()

        if self.cfg.daemon:
            util.daemonize(self.cfg.enable_stdio_inheritance)

        # set python paths
        if self.cfg.pythonpath and self.cfg.pythonpath is not None:
            paths = self.cfg.pythonpath.split(",")
            for path in paths:
                pythonpath = os.path.abspath(path)
                if pythonpath not in sys.path:
                    sys.path.insert(0, pythonpath)

        super(Application, self).run()

这个 `Application` 类则继承一个Base类 `BaseApplication`:

.. code:: python

    class BaseApplication(object):
    """
    An application interface for configuring and loading
    the various necessities for any given web framework.
    """
    def __init__(self, usage=None, prog=None):
        self.usage = usage
        self.cfg = None
        self.callable = None
        self.prog = prog
        self.logger = None
        self.do_load_config()

    def do_load_config(self):
        """
        Loads the configuration
        """
        try:
            self.load_default_config()
            self.load_config()
        except Exception as e:
            print("\nError: %s" % str(e), file=sys.stderr)
            sys.stderr.flush()
            sys.exit(1)

    def load_default_config(self):
        # init configuration
        self.cfg = Config(self.usage, prog=self.prog)

    def init(self, parser, opts, args):
        raise NotImplementedError

    def load(self):
        raise NotImplementedError

    def load_config(self):
        """
        This method is used to load the configuration from one or several input(s).
        Custom Command line, configuration file.
        You have to override this method in your class.
        """
        raise NotImplementedError

    def reload(self):
        self.do_load_config()
        if self.cfg.spew:
            debug.spew()

    def wsgi(self):
        if self.callable is None:
            self.callable = self.load()
        return self.callable

    def run(self):
        try:
            Arbiter(self).run()
        except RuntimeError as e:
            print("\nError: %s\n" % e, file=sys.stderr)
            sys.stderr.flush()
            sys.exit(1)

所以当实例化 `WSGIApplication` 的时候, 第一步做的其实是 `do_load_config()`, 这个方法中,
首先加载了默认配置 `load_default_config()`, 其实就是实例化了 `gunicorn.config.Config()` 类.
然后则是加载我们实际的配置: `load_config()`. `load_config()` 的时候做了一步 `init()` 的操作.
这个 `init()` 就是 `WSGIApplication()` 类里的 `init()` 方法. 当然, 我们可以覆写这个 `init()` 方法.
然后就是调用了 `run()` 这个方法. `run()` 方法里, 则尝试加载我们app. `util.import_app(self.app_uri)` .
最后调用 `BaseApplication` 类的 `run()` 方法. 这个 `run()` 方法中则启动了我们所谓的 `master`: `Arbiter(self).run()`.

接下来, 我们就需要进入到 `gunicorn.arbiter.Arbiter` 中一探究竟了.
