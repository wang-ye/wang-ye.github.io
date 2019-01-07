Loguru is a Python package to simplify logging setup. It provides some neat features. In this tutorial, I will discuss some of its useful features in this package.

## Simple Use Cases

```python
from loguru import logger

logger.debug("That's it, beautiful and simple logging!")
logger.add("file_1.log", rotation="500 MB")    # Automatically rotate too big file

```

## How is debug method implemented?
debug method, like info, warning and error methods, are all implemented through a decorator.

```python
# https://github.com/Delgan/loguru/blob/master/loguru/_logger.py
@staticmethod
@functools.lru_cache()
def _make_log_function(level, decorated=False):
    # Lots of code omitted ...
    def log_function(_self, _message, *args, **kwargs):
        if not _self._handlers:
            return

        # Code to build the record omitted ...
        record = {
            "elapsed": elapsed,
            "exception": exception,
            "extra": {**_self._extra_class, **_self._extra},
            "file": file_recattr,
            "function": code.co_name,
            "level": level_recattr,
            "line": frame.f_lineno,
            "message": _message,
            "module": splitext(file_name)[0],
            "name": name,
            "process": process_recattr,
            "thread": thread_recattr,
            "time": current_datetime,
        }
        # Lots of code omitted ...
        handlers = _self._handlers.copy().values()

        for handler in handlers:
            handler.emit(record, level_color, _self._ansi, _self._raw)

    doc = r"Log ``_message.format(*args, **kwargs)`` with severity ``'%s'``." % level_name
    log_function.__doc__ = doc

    return log_function
debug = _make_log_function.__func__("DEBUG")
info = _make_log_function.__func__("INFO")
```

Clearly, the code utilizes the underlying handlers for the real logging work.

## Thread and Process Safe Logging
It is worth noting that the Python's builtin logger is thread-safe by design, but not process-safe. So how does loguru achive processs-safe? The answer is using [Queue](https://docs.python.org/2/library/queue.html) provided in multiprocessing package.

When creating the handler to support process-safety, a flag called *enqueue* is also passed.

```python
# https://github.com/Delgan/loguru/blob/master/loguru/_handler.py
class Handler:
    def __init__(..., *, enqueue):
        self.lock = threading.Lock()
        if self.enqueue:
            self.queue = multiprocessing.SimpleQueue()
            self.thread = threading.Thread(target=self.queued_writer, daemon=True)
            self.thread.start()
```

If *enqueue* is set to True, then we start a process-safe queue, as well as a thread to read data from the queue. The lock is used to control the read/write to the queue. The write operation is also relatively straightforward.

```python
# https://github.com/Delgan/loguru/blob/master/loguru/_handler.py
class Handler:
    def emit(self, record, level_color, ansi_message, raw):
        str_record = build_str_record(...)
        with self.lock:
            if self.enqueue:
                self.queue.put(str_record)
            else:
                self.writer(str_record)

    def queued_writer(self):
        message = None
        queue = self.queue
        try:
            while True:
                message = queue.get()
                if message is None:
                    break
                self.writer(message)
        except _:
            if message and hasattr(message, "record"):
                message = message.record
            self.handle_error(message)
```

Is queue process safe? or thread safe?
can simplequeue work?
what does the parameter mean?
 __init__(self, maxsize=0, *, ctx):

## Is Loguru really needed?
Actually, I do not think Loguru will make a big difference. It is an alternative implementation for Python's builtin logging module, and create extra learning and maintenance burdens for people who want to use it. On the other hand, it provides some good learning materials for people willing to know more on loggers.