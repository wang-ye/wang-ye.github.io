[Flask](https://github.com/pallets/flask) supports auto-reload feature. Under auto-reload mode, once the project files change, Flask will automatically restart the app with the latest files. In this blog, I will explore how this feature is implemented.

## High-Level Idea
Flask's auto-reload feature is implemented by [Werkzeug](https://github.com/pallets/werkzeug), the German word for "tool". Werkzeug utilizes another package [Watchdog](https://github.com/gorakhargosh/watchdog) to monitor file system changes and restart when needed. Here is what happens when running with auto-reload. The **main process** first starts an endless loop. During each loop, it spawns a *Werkzeug* process with *WERKZEUG_RUN_MAIN* set to true. It is worth noting that at any time at most one Werkzeug process exists. This process will again spawn two threads - one monitoring the file changes using *Watchdog*, and the other running the actual application. Once file changes happen, the Werkzeug process is killed by a SIGQUIT signal. The next loop is then executed, creating a new *Werkzeug* process running with the latest source codes. We will dig into the actual source code in the remaining post.

## Understanding Source Code
First, let's look at the main process. Initially, the *WERKZEUG_RUN_MAIN* is not set, so the flow calls *restart_with_reloader* method, and spawns *Werkzeug* process with almost the same parameters, except setting *WERKZEUG_RUN_MAIN* to true. The Werkzeug process then runs the actual application by calling ``threading.Thread(target=main_func, args=())``.

```python
reloader_loops = {
    'auto': WatchdogReloaderLoop,
}


def _get_args_for_reloading():
    rv = [sys.executable]
    rv.append(sys.argv[0])
    rv.extend(sys.argv[1:])
    return rv


class WatchdogReloaderLoop
    # Omit other methods
    # ...
    def restart_with_reloader(self):
        """Spawn a new Python interpreter with the same arguments as this one,
        but running the reloader thread.
        """
        while 1:
            print('info', ' * Restarting with %s' % self.name)
            args = _get_args_for_reloading()
            new_environ = os.environ.copy()
            new_environ['WERKZEUG_RUN_MAIN'] = 'true'

            # Run the actual method in a different process.
            exit_code = subprocess.call(args, env=new_environ,
                                        close_fds=False)
            if exit_code != 3:
                return exit_code


def run_with_reloader(main_func, extra_files=None, interval=1,
                      reloader_type='auto'):
    """Run the given function in an independent python interpreter."""
    import signal
    reloader = reloader_loops[reloader_type](extra_files, interval)
    # Capture terminate signal gracefully.
    signal.signal(signal.SIGTERM, lambda *args: sys.exit(0))
    try:
        if os.environ.get('WERKZEUG_RUN_MAIN') == 'true':
            # The actual method we want to run!
            t = threading.Thread(target=main_func, args=())
            t.setDaemon(True)
            t.start()
            reloader.run()
        else:
            # This is the entry point of reloader. An infinite loop
            sys.exit(reloader.restart_with_reloader())
    except KeyboardInterrupt:
        pass


if __name__ == '__main__':
    run_with_reloader(your_main_method)
```

The ``reloader.run`` starts the file system monitoring. On file system changes, the *stop* signal will be issued by calling ``sys.exit(3)``. You might wonder how the ``should_reload`` is set: once file change (creation, modification, move or deletion) happens, the event handler is triggered, setting the ``should_reload`` to true. The Watchdog package is used to monitor file system changes.

```python
class WatchdogReloaderLoop(object):
    def __init__(self, ...):
        class _CustomHandler(FileSystemEventHandler):
            def on_created(self, event):
                _check_modification(event.src_path)

            def on_modified(self, event):
                _check_modification(event.src_path)

            def on_moved(self, event):
                _check_modification(event.src_path)
                _check_modification(event.dest_path)

            def on_deleted(self, event):
                _check_modification(event.src_path)
        self.event_handler = _CustomHandler

    def _check_modification(filename):
        if filename in self.extra_files:
            self.trigger_reload(filename)
        dirname = os.path.dirname(filename)
        if dirname.startswith(tuple(self.observable_paths)):
            if filename.endswith(('.pyc', '.pyo', '.py', 'txt')):
                self.trigger_reload(filename)

    # What to monitor.
    def trigger_reload(self, filename):
        # This is called inside an event handler, which means throwing
        # SystemExit has no effect.
        # https://github.com/gorakhargosh/watchdog/issues/294
        self.should_reload = True
        filename = os.path.abspath(filename)
        print('info', ' * Detected change in %r, reloading' % filename)

    # Omitted other parts.
    def run(self):
        watches = {}
        observer = self.observer_class()
        observer.start()

        try:
            while not self.should_reload:
                to_delete = set(watches)
                paths = _find_observable_paths(self.extra_files)
                for path in paths:
                    if path not in watches:
                        try:
                            watches[path] = observer.schedule(
                                self.event_handler, path, recursive=True)
                        except OSError:
                            # Clear this path from list of watches We don't want
                            # the same error message showing again in the next
                            # iteration.
                            watches[path] = None
                    to_delete.discard(path)
                for path in to_delete:
                    watch = watches.pop(path, None)
                    if watch is not None:
                        observer.unschedule(watch)
                self.observable_paths = paths
                time.sleep(self.interval)
        finally:
            observer.stop()
            observer.join()

        sys.exit(3)
```

Finally, let's see how to identify the paths to watch. The file paths consist of three parts: *sys.path*, *sys.modules* and *extrra_files*. Note that there will be tons of files in those paths, but we only need to monitor the common roots. This is achieved by an elegant algorithm in ``_find_common_roots`` which find a minimal set of root files.

```python
def _find_observable_paths(extra_files=None):
    """Finds all paths that should be observed."""
    rv = set(os.path.dirname(os.path.abspath(x))
             if os.path.isfile(x) else os.path.abspath(x)
             for x in sys.path)

    for filename in extra_files or ():
        rv.add(os.path.dirname(os.path.abspath(filename)))

    for module in list(sys.modules.values()):
        fn = getattr(module, '__file__', None)
        if fn is None:
            continue
        fn = os.path.abspath(fn)
        rv.add(os.path.dirname(fn))

    return _find_common_roots(rv)


def _find_common_roots(paths):
    """Out of some paths it finds the common roots that need monitoring."""
    paths = [x.split(os.path.sep) for x in paths]
    root = {}
    for chunks in sorted(paths, key=len, reverse=True):
        node = root
        for chunk in chunks:
            node = node.setdefault(chunk, {})
        node.clear()

    rv = set()

    def _walk(node, path):
        for prefix, child in iteritems(node):
            _walk(child, path + (prefix,))
        if not node:
            rv.add('/'.join(path))
    _walk(root, ())
    return rv
```

## Putting All Things Together
To test the auto-reload feature, I write a simple test method. It reads the file in given location, and prints the content infinitely. The latest content of the file will always appear in the terminal with the help of reloading feature.

```python
def output_strings():
    contents = open(FILE_NAME).readlines()
    while True:
        time.sleep(5)
        print(contents)
```

I put the source code into a Github [repository](https://github.com/wang-ye/code/blob/master/daily/05_01_18_output_strings_reload.py).