---
layout: post
title:  "Effective Python Notes"
date:   2017-08-05 19:59:03 -0800
---

I recently read the book [Effective Python](http://www.effectivepython.com/). The following tips are mostly from the above book, with a couple from my own experience. I categorize the tips into four buckets - Python readability, functions, classes, exceptions, concurrency and misc stuff.

## Better Python Readability
Readability is defined as "the quality of being easy or enjoyable to read.". In programming, *simple* and *maintainable* code is preferred.

**Prefer helper class to dictionaries and tuples**

When representing complex data structures, dictionaries and tuples are not descriptive enough. Helper classes wrapping the complex structures can usually make the code easier to understand and maintain.

**Avoiding complex list comprehensions**

List comprehension is super powerful. However, when the expression becomes too complex, we should refactor it with helper functions.

**No map or filter**

When comparing with list comprehension, *map* and *filter* operators are more complex due to the use of lambda. Usually we prefer list comprehension. Note that there are similar list comprehension support for map and set data structures.

**Assert on internal state often**

This does not appear in the Effective Python book, but I see its frequent use in many mission critical projects. Assertions help verify the internal state and catch the bugs *early*.

**Follow PEP8 naming rules**

1. For private var, use names like *__double_leading_underscore*.
2. Specify the package when importing the package. Use absolute import if possible. If  you must do relative imports, use the explicit syntax 
`from . import foo`.

## Exception Handling

**The power of else in exception handling**

The *else* statement in Python exception helps separate the normal execution with exception handling. Good to use more often!

**Prefer exceptions to returning None**

This is a rule from the book, but I feel returning None or Exception has both pros and cons.
Returning exception is sometimes better because None and other values (e.g., zero, the empty string) all evaluate to False in conditional expressions.
However, when using exceptions, more efforts need to be taken. Methods throwing exception needs to be documented clearly, and caller needs to handle exception.

**Define root exceptions for your API**

It is better to define custom exceptions with hierarchy for your API.
Several advantages:

1. It helps API consumer to easily capture the exceptions triggered by your API.
2. It helps reveal bugs in your API. If your code only deliberately raises  exceptions that you define  within your module’s hierarchy, then all other   types of exceptions raised by your module must be the ones that you didn’t  intend to raise. These are bugs.
3. The third impact of using root exceptions is future-proofing your API. API owner can subclass the Exceptions without affecting API consumers.

## Function tricks
**Avoid complex closures**

Define classes with __call__ to store states instead of complex closures. For example, you have a complex sorting requirement requiring state management. In this case, class with __call__ will be cleaner.

**Set dynamic keyword arguments to None**

When dealing with dynamic keyword arguments, set the default value to None, and use docstring to specify the possible values.

## Class and Objects
**Regarding Python getter and setter**

For simple getter and setter scenarios, just use public attribute. 
If we have complex getter or setter requirement, i.e., validating the data when setting, consider using @property and property.setter.

**Multi-inheritance**

Multiple inheritance is rarely needed. If you really want to use it, consider *mixins* to create complex functionality from simple behaviors. Always use super() to initialize parent classes as it has better treatment to multi-inheritance.

**metaclass**

We rarely need it ... If you think you need it, think again ... If you really need it, then probably you already know how to use it.

**When to use @classmethod**

Python does not support constructor overloading. @classmethod can be used to simulate this behavior. Also, note that Python also works well with [inheritance](https://www.programiz.com/python-programming/methods/built-in/classmethod).

## Concurrency and Parallelism
**Use Threads for Blocking I/O, avoid for Parallelism**

Due to GIL, threads should only be used for blocking I/O.

**Use concurrent.future for true parallelism**

concurrent.future provides a simpler interface for true parallelism. If it still does not fit your needs, consider using multiprocessing module directly.

## Python Misc
**Generators and List Comprehension**

For generator, be careful when iterating multiple times as the first iteration will deplete the generator.

**Unicode in Python 3**

<!-- Bytes, string and Unicode -->
<!-- Understanding unicode, bytes and str -->
It is important to understand the difference between bytes and str.
bytes is *raw 8-bit values*, while str is *Unicode characters*. In Python 3, byte var can never equal to str var.
Unicode is a way to assign unique number to every character in the world, while utf-8 is a way to represent the Unicode with sequences of 8-bit bytes.
<!-- In  Python  2,  there   are two types   that    represent   sequences   of  characters: str and unicode.    In  contrast    to  Python  3,  instances   of  str contain raw 8-bit   values. Instances   of unicode contain Unicode characters. -->
A caveat: when writing binary data, open file with *'+w'*.
<!-- more strict -->

**__repr__ VS __str__**

[In short](https://stackoverflow.com/questions/1436703/difference-between-str-and-repr-in-python),
\__repr__ to be unique, and \__str__ to be readable. Implementing \__repr__ should be the second nature.
<!-- So one should first write a __repr__ that allows you to reinstantiate an equivalent object from the string it returns e.g. using eval or by typing it in character-for-character in a Python shell. -->
