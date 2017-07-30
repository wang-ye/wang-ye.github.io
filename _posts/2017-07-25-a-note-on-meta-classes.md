On A Metaclasses Use Case

metaclass is one of the black magics in Python. In this blog, I discusses two use cases for it.

## What is Metaclass?
There is a very through [stackoverflow discussion](https://stackoverflow.com/a/6581949) regarding metaclass. Here I am giving a quick summary:

Classes can create instances. What's more interesting is, classes are themselves instances of *metaclasses*.

The *type* keyword is what makes the magic happens. It can be viewed as a metaclass, or a class factory. You can create classes by using this keyword. Here I steal some contents from the [stackoverflow discussion](https://stackoverflow.com/a/6581949):

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

The metaclass, after all, is a class that creates other classes. *type* is a built-in metaclass. You can create customized metaclass.

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

Classes are objects.
You can create dynamic classes.
metaclass is to create classes dynamically.
everything is an object?

## Use Case 1: Fake Type Proxy
[freezegun](https://github.com/spulec/freezegun) allows the code the freeze the time. Once the decorator have been invoked, all calls to datetime.datetime.now(), datetime.datetime.utcnow(), datetime.date.today() will return the time that has been frozen. Under the hood, it uses metaclass to create a FakeDateTime that behaves like DateTime.

```python
from freezegun import freeze_time
import datetime


# instancecheck magic
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

Basically, the code snippet builds a FakeDatetime class. It contains almost the same method interfaces as Datetime, except that each call will return a previously frozen date. I added the push_time and pop_time method to explicitly manage the stack FakeDatetime maintains. Basically, after you push a frozen time to the stack, the *now*, *utcnow* and *today* will all return the last pushed time.

FakeDatetime is a thin wrapper which implements all the key methods. Metaclass is here to elegantly override the isinstance method. A datetime.datetime object will be considered as an instance of FakeDatetime.

So why is it implemented this way? Why cannot FakeDatetime just inherit datetime.datetime? It can, but then a lot more methods need to be overridden, and it is not as clean as the this metaclass approach. A similar discussion about isinstance override is [here](https://stackoverflow.com/questions/6803597/how-to-fake-type-with-python).

## Use Case 2: Class Validation
In the book [Effective Python](), the author provides another interesting use case in **Item 34: Register class existence with Metaclass**. Imagine that you want to implement a customized JSON serializer/deserializer, and you want this to work for a set of classes. Basically, we want to implement two methods:

```python
# Convert the params in JSON format.
def serialize(params):
    pass

# Build a object with the previously serialized data.
def deserialize(serialized_object):
    pass
```

Modify class creation behaviors. During the class creation, you can register class, verify the parameters of the class and other things.

Ideally, we want each class to have a method serialize() converting input params to a JSON blob. Also, we will have a global method that deserialize *any* JSON blob to the corresponding object. This means we must also attach the class information in the JSON blob. Note that all the code snippets are copied from the the Effective Python book

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

Metaclass is thus introduced to conduct class registration at class creation time.

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
metaclass is the factory for classes. It is one of the deeply hidden Python dark magic, and should not be used in 99% of the scenarios. However, it can also be super powerful for certain problems. This blog introduces two of them. You can also find more use cases such as class validation and annotation in [this awesome book]().