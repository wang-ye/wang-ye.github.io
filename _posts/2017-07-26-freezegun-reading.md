---
layout: post
title:  "Understanding Freezegun Internals"
date:   2017-07-26 20:49:03 -0800
---
Today I want to share some learnings when reading [freezegun](https://github.com/spulec/freezegun). From the author, FreezeGun *is a library that allows your python tests to travel through time by mocking the datetime module*. Once the decorator or context manager have been invoked, all calls to datetime.datetime.now(), datetime.datetime.utcnow(), datetime.date.today(), time.time(), time.localtime(), time.gmtime(), and time.strftime() will return the time that has been frozen.

Freezegun supports the following features:

1. It provides a decorator and the context manager to conduct time travel (examples will follow later).
2. It has a manual tick mode, which can move the time forward with fixed seconds.
3. It can move time to a fixed datetime.

In this blog, I want to do something different - a minimal set of code that implements the above core features will be extracted and discussed, rather than the original repository. To simplify a bit, I will only consider using datetime module.

## Some Usage Examples
The following snippet illustrates the key freezegun usages:

```python
import datetime
import unittest

# Use decorator.
@freeze_time("2012-01-14")
def decorator_test():
 assert datetime.datetime.now() == datetime.datetime(2012, 1, 14)

# Use context manager.
def context_manager_test():
    assert datetime.datetime.now() != datetime.datetime(2012, 1, 14)
    with freeze_time("2012-01-14"):
        assert datetime.datetime.now() == datetime.datetime(2012, 1, 14)
    assert datetime.datetime.now() != datetime.datetime(2012, 1, 14)

# Test time move_to feature.
def test_move_to():
    initial_datetime = datetime.datetime(year=1, month=7, day=12,
                                        hour=15, minute=6, second=3)

    other_datetime = datetime.datetime(year=2, month=8, day=13,
                                        hour=14, minute=5, second=0)
    with freeze_time(initial_datetime) as frozen_datetime:
        assert frozen_datetime() == initial_datetime

        frozen_datetime.move_to(other_datetime)
        assert frozen_datetime() == other_datetime

        frozen_datetime.move_to(initial_datetime)
        assert frozen_datetime() == initial_datetime
```

Assume we want to support these use cases, how should we design this *freeze_time* method?

## A Minimal Implementation
First, we have to pass in a frozen datetime and store it somewhere. let's think about the data structure to store it. Besides being very similar to a *datetime.datetime* object, it also needs to support *move_to* and *ticker* method. Freezegun uses an elegant way by defining class with *__call__* method.

```python
class FrozenDateTimeFactory(object):
    def __init__(self, time_to_freeze):
        self.time_to_freeze = time_to_freeze

    def __call__(self):
        return self.time_to_freeze

    def tick(self, delta=datetime.timedelta(seconds=1)):
        if isinstance(delta, numbers.Real):
            self.time_to_freeze += datetime.timedelta(seconds=delta)
        else:
            self.time_to_freeze += delta

    def move_to(self, target_datetime):
        """Move frozen date to the given ``target_datetime."""
        target_datetime = _parse_time_to_freeze(target_datetime)
        delta = target_datetime - self.time_to_freeze
        self.tick(delta=delta)
```

So after calling *frozen_obj = FrozenDateTimeFactory(time_to_freeze)*, we can easily get the original time_to_freeze object by simply calling *frozen_obj()*. What's more, *tick* and *move_to* can also be invoked by *frozen_obj*. What a powerful trick!

Next, let's think about how we can freeze the time. Basically,  We need a **context** to temporarily change datetime.datetime to a fake class, which will return the frozen object when calling methods like *now* and *utcnow*.
When the tests are done, the *context* can change the datetime back to normal. So context manager comes to rescue! The code needs to have a class with __enter__ and __start__. We store the frozen datetime in a stack to enable nested usage. The detailed snippet:

```python
class FreezeTimeInternal(object):
    def __init__(self, time_to_freeze_str, tz_offset):
        self.time_to_freeze = _parse_time_to_freeze(time_to_freeze_str)
        self.tz_offset = tz_offset

    def __enter__(self):
        return self.start()

    def __exit__(self, *args):
        self.stop()

    def start(self):
        time_to_freeze = FrozenDateTimeFactory(self.time_to_freeze)
        datetime.datetime = FakeDatetime
        datetime.datetime.times_to_freeze.append(time_to_freeze)
        datetime.datetime.tz_offsets.append(self.tz_offset)
        return time_to_freeze

    def stop(self):
        datetime.datetime.times_to_freeze.pop()
        datetime.datetime.tz_offsets.pop()
        if not datetime.datetime.times_to_freeze:
            datetime.datetime = real_datetime    
```

Here in the start of a context, we replace datetime.datetime with a new class called FakeDatetime, which provides almost the same interface as a normal datetime object. The only difference is, it will only return the dates of the frozen objects. Here is the *FakeDatetime*.

```python
real_datetime = datetime.datetime
class FakeDatetime(with_metaclass(FakeDatetimeMeta, real_datetime)):
    times_to_freeze = []
    tz_offsets = []

    def __new__(cls, *args, **kwargs):
        return real_datetime.__new__(cls, *args, **kwargs)

    def __add__(self, other):
        result = real_datetime.__add__(self, other)
        if result is NotImplemented:
            return result
        return datetime_to_fakedatetime(result)

    def __sub__(self, other):
        result = real_datetime.__sub__(self, other)
        if result is NotImplemented:
            return result
        if isinstance(result, real_datetime):
            return datetime_to_fakedatetime(result)
        else:
            return result

    @classmethod
    def now(cls, tz=None):
        now = cls._time_to_freeze() or real_datetime.now()
        if tz:
            result = tz.fromutc(now.replace(tzinfo=tz)) + datetime.timedelta(hours=cls._tz_offset())
        else:
            result = now + datetime.timedelta(hours=cls._tz_offset())
        return datetime_to_fakedatetime(result)

    @classmethod
    def today(cls):
        return cls.now(tz=None)

    @classmethod
    def utcnow(cls):
        result = cls._time_to_freeze() or real_datetime.utcnow()
        return datetime_to_fakedatetime(result)

    @classmethod
    def _time_to_freeze(cls):
        if cls.times_to_freeze:
            return cls.times_to_freeze[-1]()

    @classmethod
    def _tz_offset(cls):
        return cls.tz_offsets[-1]

FakeDatetime.min = datetime_to_fakedatetime(real_datetime.min)
FakeDatetime.max = datetime_to_fakedatetime(real_datetime.max)
```

I do not quite get this *with_metaclass* call, but eventually FakeDatetime supports the same interface as datetime.datetime, and it stores the frozen time in a stack called *_times_to_freeze*. Now the whole flow seems reasonable!

Wait, I forgot the decorator part and freeze_time glue code? Here you go!

```python
class FreezeTimeInternal(object):
    ...
    def __call__(self, func):
        return self.decorate_callable(func)

    def decorate_callable(self, func):
        def wrapper(*args, **kwargs):
            # Calling context manager.
            with self:
                result = func(*args, **kwargs)
            return result
        functools.update_wrapper(wrapper, func)

        # update_wrapper already sets __wrapped__ in Python 3.2+, this is only
        # needed for Python 2.x support
        wrapper.__wrapped__ = func
        return wrapper

def freeze_time(time_to_freeze, tz_offset=0):
  return FreezeTimeInternal(time_to_freeze, tz_offset)
```

The decorator is basically calling the context manager to implement the time travel. freeze_time takes frozen time, and pass it to FreezeTimeInternal.

## Summary
This blog briefly discusses the internals of freezegun, which provides an elegant solution for resembling some built-in classes (datetime and time). I created a [working snippet](https://github.com/wang-ye/code/blob/master/python/date_test.py) for your reference. The full freezegun implementation can be found [here](https://github.com/spulec/freezegun).