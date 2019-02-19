I was recently setting up [Sentry](https://sentry.io/) for a simple Flask app to catch the potential errors, and was impressed by both the simplicity and power of the sentry setup - with only a couple of lines to integrate with the Flask sentry sdk, Sentry was able to catch all the errors and logs. This led me to explore how Sentry flask sdk works. The following was the findings while reading the source codes.

## Setup Sentry for Flask App
The sentry setup is unbelievably simple:

```python
# In __init__.py
from flask import Flask
import sentry_sdk

from sentry_sdk.integrations.flask import FlaskIntegration
sentry_sdk.init(dsn="your_dsn_name", integrations=[FlaskIntegration()])

app = Flask(__name__)
```

From [Sentry documentations](https://docs.sentry.io/platforms/python/flask/#behavior), sentry-flask integrations have the following key characteristics:

1. All exceptions leading to an Internal Server Error are reported, even it is captured and handled already.
2. Request data is attached to all events: HTTP method, URL, headers, form data, JSON payloads.
3. Logging with app.logger or any logger will create breadcrumbs when the Logging integration is enabled (done by default)

In this post we will focus on 1&3. The key techniques used here, as explained by the doc, is [Flask Signals](http://flask.pocoo.org/docs/1.0/signals/). In the rest of the post, I will first discuss the Flask signals and how it can help capture the potential errors, and then discuss how Sentry plugin implements feature one and three.

### Flask Signals

From the original doc,

> Flask comes with a couple of signals and other extensions might provide more. Also keep in mind that signals are intended to notify subscribers and should not encourage subscribers to modify data. You will notice that there are signals that appear to do the same thing like some of the builtin decorators do (eg: request_started is very similar to before_request()). However, there are differences in how they work. The core before_request() handler, for example, is executed in a specific order and is able to abort the request early by returning a response. In contrast all signal handlers are executed in undefined order and do not modify any data.

As a more concrete example, let's take a look at the code provided by Flask:

```python
from flask import template_rendered
from contextlib import contextmanager


@contextmanager
def captured_templates(app):
    recorded = []
    def record(sender, template, context, **extra):
        recorded.append((template, context))
    template_rendered.connect(record, app)
    try:
        yield recorded
    finally:
        template_rendered.disconnect(record, app)


# This can now easily be paired with a test client:
with captured_templates(app) as templates:
    rv = app.test_client().get('/')
    assert rv.status_code == 200
    assert len(templates) == 1
    template, context = templates[0]
    assert template.name == 'index.html'
    assert len(context['items']) == 10
```

The above code uses Flask signals to catch the "template render" events and check the right template has been rendered. For Sentry, Flask signal is something to subscribe to, which facilitates sending the corresponding events and breadcrumbs.

## Catching Exceptions in Sentry SDK
To get started, let's take a deeper look at the ``init`` method.

```python
# sentry_sdk/hub.py
def init(*args, **kwargs):
    """Initializes the SDK and optionally integrations.

    This takes the same arguments as the client constructor.
    """
    global _initial_client
    client = Client(*args, **kwargs)
    Hub.current.bind_client(client)
    rv = _InitGuard(client)
    if client is not None:
        _initial_client = weakref.ref(client)
    return rv
```

The `init` method creates the client and assign it to the current Hub. Hub is very similar to Flask app context - both uses extensively the threading.local to provide multithreading support. In a nutshell, the `local` helps create isolated threading environment and each thread sees its own Hub object. The current property is used to get the Hub the running thread owns. For now, we can just ignore this and focus on the actual exception/logging event handling.

When initializing the client, we need to pass the integration object. So what does the FlaskIntegration do? It essentially connects with the flask exception signals and sends the events to Sentry server. Let's see how this is implemented.

```python
# in integrations/flask.py
from flask.signals import (
    got_request_exception,
)

class FlaskIntegration(Integration):
    identifier = "flask"

    transaction_style = None

    def __init__(self, transaction_style="endpoint"):
        TRANSACTION_STYLE_VALUES = ("endpoint", "url")
        if transaction_style not in TRANSACTION_STYLE_VALUES:
            raise ValueError(
                "Invalid value for transaction_style: %s (must be in %s)"
                % (transaction_style, TRANSACTION_STYLE_VALUES)
            )
        self.transaction_style = transaction_style

    @staticmethod
    def setup_once():
        # Lots of code omitted ...
        got_request_exception.connect(_capture_exception)

        old_app = Flask.__call__

        def sentry_patched_wsgi_app(self, environ, start_response):
            if Hub.current.get_integration(FlaskIntegration) is None:
                return old_app(self, environ, start_response)

            return SentryWsgiMiddleware(lambda *a, **kw: old_app(self, *a, **kw))(
                environ, start_response
            )

        Flask.__call__ = sentry_patched_wsgi_app

def _capture_exception(sender, exception, **kwargs):
    hub = Hub.current
    if hub.get_integration(FlaskIntegration) is None:
        return
    event, hint = event_from_exception(
        exception,
        client_options=hub.client.options,
        mechanism={"type": "flask", "handled": False},
    )

    hub.capture_event(event, hint=hint)
```

The ``got_request_exception`` is used by Flask to signal exception happening. The FlaskIntegration utilizes this for the actual notification. Also, due to the use of ``sentry_patched_wsgi_app``, we know why Flask initialization should happen after the sentry sdk initialization.
You may also notice that, there are no disconnect called for the events. Clearly there is no need - the sentry notification is supposed to have the same lifecycle of the app and thus there is no need for the *disconnect*.

## Capturing Logs
Error logs, on the other hand, are handled with monkeypatch on the internal logging handler.

```python
import logging
from sentry_sdk.integrations import Integration

DEFAULT_LEVEL = logging.INFO
DEFAULT_EVENT_LEVEL = logging.ERROR

_IGNORED_LOGGERS = set(["sentry_sdk.errors"])


class LoggingIntegration(Integration):
    identifier = "logging"

    def __init__(self, level=DEFAULT_LEVEL, event_level=DEFAULT_EVENT_LEVEL):
        self._handler = None
        self._breadcrumb_handler = None

        if level is not None:
            self._breadcrumb_handler = BreadcrumbHandler(level=level)

        if event_level is not None:
            self._handler = EventHandler(level=event_level)

    def _handle_record(self, record):
        if self._handler is not None and record.levelno >= self._handler.level:
            self._handler.handle(record)

        if (
            self._breadcrumb_handler is not None
            and record.levelno >= self._breadcrumb_handler.level
        ):
            self._breadcrumb_handler.handle(record)

    @staticmethod
    def setup_once():
        old_callhandlers = logging.Logger.callHandlers

        def sentry_patched_callhandlers(self, record):
            try:
                return old_callhandlers(self, record)
            finally:
                # This check is done twice, once also here before we even get
                # the integration.  Otherwise we have a high chance of getting
                # into a recursion error when the integration is resolved
                # (this also is slower).
                if record.name not in _IGNORED_LOGGERS:
                    integration = Hub.current.get_integration(LoggingIntegration)
                    if integration is not None:
                        integration._handle_record(record)

        logging.Logger.callHandlers = sentry_patched_callhandlers
```

So in the ``setup_once`` method, the ``callHandlers`` method is patched with some extra actions - sending logs and breadcrumbs to Sentry. The `_handle_record` here utilizes two internal handlers BreadcrumbHandler and EventHandler for sending events to Sentry. I omitted the details for simplicity.

## Summary
Sentry's Flask SDK illustrates two common ways to intercept error messages and send them back to Sentry. One approach is with signals to catch all the exceptions received by Flask, while the other way is monkeypatching the internal callHandlers method in logging module for sending the log messages. With their help we can now have a three-line sentry configuration while having all the errors captured and sent to Sentry automatically.