I encountered a wired error when playing with Flask. Flask provides ``flask run`` command to easily launch a new server. To do so, you just need to set an environment variable ``FLASK_APP``.

```shell
export FLASK_APP=myserver.py
```

Here is the project structure:
```
projectfolder
    __init__.py
    app
        __init__.py
        models.py
        templates
            ...
        routes.py
    myserver.py
```

After setting ``FLASK_APP``, surprisingly the ``flask run`` command failed with the *NoAppException* error. The detail is as follows:

```
flask.cli.NoAppException
flask.cli.NoAppException: While importing your project, an ImportError was raised:

Traceback (most recent call last):
  File "/Users/ye/github/code/python/myserver/venv/lib/python3.6/site-packages/flask/cli.py", line 238, in locate_app
    __import__(module_name)
  File "/Users/ye/github/myproject/myserver.py", line 7, in <module>
    from app import app, db
ModuleNotFoundError: No module named 'app'
```

What happened here? When import error happens, the first debugging step is checking the ``sys.path`` variable of the imported file. It turned out that the *myserver.py* path is not included in ``sys.path``.
After digging into the source code, we can find how the flask run command appends the ``myserver.py`` to ``sys.path``. The relevant code is listed below:

```python
# In cli.py
def prepare_import(path):
    """Given a filename this will try to calculate the python path, add it
    to the search path and return the actual module name that is expected.
    """
    path = os.path.realpath(path)

    if os.path.splitext(path)[1] == '.py':
        path = os.path.splitext(path)[0]

    if os.path.basename(path) == '__init__':
        path = os.path.dirname(path)

    module_name = []

    # move up until outside package structure (no __init__.py)
    while True:
        path, name = os.path.split(path)
        module_name.append(name)

        if not os.path.exists(os.path.join(path, '__init__.py')):
            break

    if sys.path[0] != path:
        sys.path.insert(0, path)

    return '.'.join(module_name[::-1])

def load_app(self):
    """Loads the Flask app (if not yet loaded) and returns it.  Calling
    this multiple times will just result in the already loaded app to
    be returned.
    """
    __traceback_hide__ = True

    if self._loaded_app is not None:
        return self._loaded_app

    app = None

    if self.create_app is not None:
        app = call_factory(self, self.create_app)
    else:
        if self.app_import_path:
            path, name = (self.app_import_path.split(':', 1) + [None])[:2]
            import_name = prepare_import(path)
            app = locate_app(self, import_name, name)
        else:
            for path in ('wsgi.py', 'app.py'):
                import_name = prepare_import(path)
                app = locate_app(self, import_name, None,
                                 raise_if_not_found=False)

                if app:
                    break
```

Now the root cause is clear - because ``projectfolder`` also has a ``__init__.py`` file, it is considered a module. Flask mistakenly inserted into ``sys.path`` the parent folder of ``projectfolder`` rather than itself. This causes Flask app failing to locate the server file correctly.

*The remedy*: Remove the ``__init__.py`` file under projectfolder directory!