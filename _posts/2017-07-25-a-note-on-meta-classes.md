---
layout: post
title:  "On Metaclass Use Cases"
date:   2017-07-25 20:49:03 -0800
---
Metaclass is one of the powerful black magics in Python. In this blog, I will discuss what it is, and illustrate its strength two interesting use cases.

## What is Metaclass?
There is a very through [stackoverflow discussion](https://stackoverflow.com/a/6581949) regarding metaclass. Here I am giving a quick summary based on that.
As is known to all, classes can create instances. But not so many people know that classes are themselves instances of *metaclasses*. 
The metaclass, after all, is a class that creates other classes. *type* is the built-in metaclass, which you can use to create customized metaclasses. Here is the basic use of *type*:

```python
# type usage:
# type(name of the class, 
#      tuple of the parent class (for inheritance, can be empty), 
#      dictionary containing attributes names and values)

>>> class MyShinyClass(object):
...       pass
# can be created manually this way:

>>> MyShinyClass = type('MyShinyClass', (), {}) # returns a class object
>>> print(MyShinyClass)
<class '__main__.MyShinyClass'>
>>> print(MyShinyClass()) # create an instance with the class
<__main__.MyShinyClass object at 0x8997cec>
```

As you can see, a class created with class keyword can be *equivalently* constructed with *type* keyword. *type* thus can be viewed as a class factory -- by passing different parameters, you create different classes with it. You can also customize with *type*:

```python
class UpperAttrMetaclass(type): 

    def __new__(cls, clsname, bases, dct):
        uppercase_attr = {}
        for name, val in dct.items():
            if not name.startswith('__'):
                uppercase_attr[name.upper()] = val
            else:
                uppercase_attr[name] = val

        return super(UpperAttrMetaclass, cls).__new__(cls, clsname, bases, uppercase_attr)
```

UpperAttrMetaclass is a customized metaclass. For any class extending it, it converts all its methods to upper cases, as illustrated here:

```python
__metaclass__ = upper_attr # this will affect all classes in the module

class Foo(): # global __metaclass__ won't work with "object" though
  # but we can define __metaclass__ here instead to affect only this class
  # and this will work with "object" children
  bar = 'bip'

print(hasattr(Foo, 'bar'))
# Out: False
print(hasattr(Foo, 'BAR'))
# Out: True

f = Foo()
print(f.BAR)
# Out: 'bip'
```

OK! With basic metaclass knowledge, let's study a couple of use cases.

## Use Case 1: Fake Type Proxy
[freezegun](https://github.com/spulec/freezegun) is a package that allows testing code to freeze (yay!) time. It provides a decorator. Once the decorator is invoked with a fixed time, all calls to datetime.datetime.now(), datetime.datetime.utcnow(), datetime.date.today() will return it. Under the hood, it utilizes metaclass to create a lightweight FakeDateTime class that behaves like DateTime. In a nutshell, freezegun builds FakeDatetime like this:

```python
from freezegun import freeze_time
import datetime


def with_metaclass(meta, *bases):
    """Create a base class with a metaclass."""
    return meta("NewBase", bases, {})


# Override isinstance.
class FakeDatetimeMeta(type):
    @classmethod
    def __instancecheck__(cls, obj):
        return isinstance(obj, real_datetime)


real_datetime = datetime.datetime
# Ignore timezone effect for simplicity.
class FakeDatetime(with_metaclass(FakeDatetimeMeta, real_datetime)):
    times_to_freeze = []

    def __new__(cls, *args, **kwargs):
        return real_datetime.__new__(cls, *args, **kwargs)

    @classmethod
    def now(cls):
        now = cls._time_to_freeze() or real_datetime.now()
        return datetime_to_fakedatetime(now)

    @classmethod
    def today(cls):
        return cls.now(tz=None)

    @classmethod
    def utcnow(cls):
        result = cls._time_to_freeze() or real_datetime.utcnow()
        return datetime_to_fakedatetime(result)

    # Skip some other methods that simulate DateTime class.

    @classmethod
    def _time_to_freeze(cls):
        if cls.times_to_freeze:
            return cls.times_to_freeze[-1]()

    @classmethod
    def push_time(cls, frozen_time):
        cls._times_to_freeze.append(frozen_time)

    @classmethod
    def pop_time(cls):
        cls._times_to_freeze.pop()
```

Basically, the code snippet builds a FakeDatetime class. It contains almost the same method interfaces as Datetime, except that each call will return a previously frozen date. I added the push_time and pop_time method to explicitly manage the _times_to_freeze stack, which is useful for nested usage.

FakeDatetime is actually a thin wrapper implementing several key datetime class methods. Metaclass is here to elegantly override the *isinstance* method. With this override, datetime object will be considered an FakeDatetime instance when calling *isinstance*.

So why is it implemented this way? Can FakeDatetime just extend datetime and automatically gain all methods of datetime? It can, but then a lot more methods need to be overridden, and it is not as clean as the this **thin wrapper + metaclass** approach.

## Use Case 2: Class Registration
In the book [Effective Python](http://www.effectivepython.com/), the author provides another interesting use case in **Item 34: Register class existence with Metaclass**. Imagine you want to implement a customized JSON serializer/deserializer for a set of classes. Each class here is a data container, and the data is immutable since creation. For serializing, we only need to serialize the parameters passed to initialization. The deserializing method returns object rather than the parameters.

```python
# Convert the object params in JSON format.
def serialize(object_params):
    pass

# Build a object with the previously serialized data.
def deserialize(serialized_object):
    pass
```

We want each newly created class to have a instance method serialize() converting input parameterss to a JSON blob. Also, we will have a global method that deserialize *any* JSON blob to an object. Thus, we must attach class information in JSON blob.

```python
class Vector3D(RegisteredSerializable):
  def __init__(self,  x,  y,  z):
    super().__init__(x, y,  z)
    self.x, self.y, self.z  =   x,  y,  z

v3 = Vector3D(10, -7, 3)
print('Before:              ',  v3)
data    =   v3.serialize()
print('Serialized:',    data)
print('After:               ',  deserialize(data))
```

How can we implement this serializer/deserializer? The major challenge here is automatically maintaining an *class_name: class* mapping so that later the deserializer can get the actual class details from the class names.

Metaclass is introduced to achieve the above goal. It modifies class creation behaviors and conducts class registration during the creation. Deserializer can later correctly identify the actual class and construct the objects. Here is the code snippet:

```python
registry = {}

def register_class(target_class):
  registry[target_class.__name__] =   target_class

def deserialize(data):
  params = json.loads(data)
  name = params['class']
  target_class = registry[name]

class BetterSerializable(object):
  def __init__(self,  *args):
    self.args   =   args

  def serialize(self):
    return  json.dumps(
      {
        'class': self.__class__.__name__,
        'args': self.args,
       }
     )

  def __repr__(self):
    ...

class Meta(type):
  def __new__(meta,   name,   bases,  class_dict):
    cls = type.__new__(meta,  name,   bases,  class_dict)
    # This is the real meat for using Meta - automatic class registration during class creation.
    register_class(cls)
    return cls

class RegisteredSerializable(BetterSerializable, metaclass=Meta):
  pass
```

## Summary
metaclass is the factory for building classes. It leverage the keyword *type* for all the magics. In fact, it is a deeply hidden dark magic, and should not be used in 99% of the scenarios. However, if used correctly, it can also be super powerful. This blog introduces two use cases. You can also find other uses such as class validation and annotation in [this awesome book](http://www.effectivepython.com/).