Recently I went over Python metaclass again, and realized that metaclass-based approach shares lots of similarities with the decorator-based one. In fact, many metaclass implementations can be similarly done with the more explicit decorator approach. In this post, I will examine the registry pattern implementations with metaclass and decorator.

First, the decorator approach:

```python
class PluginRegistry(object):
    plugins = []

    @staticmethod
    def add(func):
        if func not in PluginRegistry.plugins:
            PluginRegistry.plugins.append(func)


def plugin(func):
    """Class decorator for adding plugins to the registry"""
    PluginRegistry.add(func)
    return func


@plugin
def m1():
    print('m1')


@plugin
def m2():
    print('m2')


def main():
    pr = PluginRegistry.plugins
    for c in pr:
        print(c)


if __name__ == '__main__':
    main()
```

Now, the metaclass-based approach:

```python
# Run with metaclass.
class PluginRegistry(object):
    plugins = []

    @staticmethod
    def add(func):
        if func not in PluginRegistry.plugins:
            PluginRegistry.plugins.append(func)


class MetaRegistry(type):

    def __new__(meta, name, bases, class_dict):
        cls = type.__new__(meta, name, bases, class_dict)
        if name not in PluginRegistry.plugins:
            PluginRegistry.add(cls)
        return cls


class BaseClass(object):
    __metaclass__ = MetaRegistry
    pass


class Foo(BaseClass): pass


class Bar(BaseClass): pass


def main():
    print(PluginRegistry.plugins)


if __name__ == '__main__':
    main()
```

Choosing which one to use is more of a personal preference. For decorator, developers have to decorate the method or class explicitly, but this also 
 makes the class modifications more explicit. In general, I feel the decorator approach is more readable.