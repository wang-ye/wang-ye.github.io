---
layout: post
title:  "Understanding Python Faker Implementation"
date:   2017-05-20 17:49:03 -0800
---

[Faker](https://github.com/joke2k/faker) is a powerful tool for generating fake data. Say you want to bootstrap your development database with some fake data, then faker may be your best bet. In this post we will dive into its source code and learn some powerful Python meta-programming techniques.

The package utilizes meta programming a lot. A good learning example.
To start with, some basic use cases are:

```python
# A couple of examples:
from faker import Faker
fake = Faker()

fake.name()
# 'Lucy Cechtelar'
fake.name()
# 'Lily Wolf'
fake.address()
# "426 Jordy Lodge
#  Cartwrightshire, SC 88120-6700"
```

Note that the *name* call will return a randomized result. The first thing that jumps into our mind is, what is this "Faker" thing? As it turns out, it is *create* method of faker.factory class.

```python
# faker/__init__.py
VERSION = '0.7.7'

from faker.generator import Generator
from faker.factory import Factory

Faker = Factory.create

"""
To use Faker package, you can simply call
Faker().first_name()
"""
```

The **create** method takes in locales, and builds an object (generator) to hold the methods on the fly to generate fake data.

```python
# faker/factory.py
class Factory(object):
    @classmethod
    def create(cls, locale=None, providers=None, generator=None, **config):
        # fix locale to package name
        locale = locale.replace('-', '_') if locale else DEFAULT_LOCALE
        locale = pylocale.normalize(locale).split('.')[0]
        if locale not in AVAILABLE_LOCALES:
            msg = 'Invalid configuration for faker locale "{0}"'.format(locale)
            raise AttributeError(msg)
        providers = providers or PROVIDERS
        faker = generator or Generator(**config)

        for prov_name in providers:
            if prov_name == 'faker.providers':
                continue

            # Choose provider with right locale.
            prov_cls, lang_found = cls._get_provider_class(prov_name, locale)
            provider = prov_cls(faker)
            provider.__provider__ = prov_name
            provider.__lang__ = lang_found
            # The real trick to set methods of the provider to faker.
            faker.add_provider(provider)
        return faker
```

From the implementation, the **create** method returns a Generator object.  Locale is used to choose the right provider, and we will discuss that later. The key piece is the **add_provider** method. It dynamically set the class and static methods in each provider to the generator object.

The generator is basically a holder of providers and configurations. We cannot see any methods for fake data generation associated with it. The real meat, however, lies in **add_provider** method.

```python
# faker/generator.py
class Generator(object):

    __config = {}

    def __init__(self, **config):
        self.providers = []
        self.__config = dict(
            list(self.__config.items()) + list(config.items()))

    def add_provider(self, provider):
        if type(provider) is type:
            provider = provider(self)

        self.providers.insert(0, provider)

        # Get all public class and static methods.
        for method_name in dir(provider):
            # skip 'private' method
            if method_name.startswith('_'):
                continue

            faker_function = getattr(provider, method_name)

            if hasattr(faker_function, '__call__') or \
                    isinstance(faker_function, (classmethod, staticmethod)):
                # add all faker method to generator
                self.set_formatter(method_name, faker_function)

    def set_formatter(self, name, method):
        setattr(self, name, method)
```

The *add_provider* comments here are self-explanatory: the function sets each public class and static function as an attribute of generator, so that calling methods like "first_name" and "address" is possible.

Providers, on the other hand, are all in providers directory. Each provider has a Provider class, which defines the interfaces. The locale-specific data are stored inside each locale directory.
The full list of providers: https://faker.readthedocs.io/en/latest/providers.html.

The implementation for fake generation lies in provider_name/__init__.py
Take address provider as an example:

```python
# faker/providers/address/__init__.py
class Provider(BaseProvider):
    city_suffixes = ['Ville', ]
    city_formats = ('{{first_name}} {{city_suffix}}', )
    postcode_formats = ('#####', )
    countries = [tz['name'] for tz in date_time.Provider.countries]
    country_codes = [tz['code'] for tz in date_time.Provider.countries]

    def city(self):
        """
        :example 'Sashabury'
        """
        pattern = self.random_element(self.city_formats)
        return self.generator.parse(pattern)

    # Many other class method definitions.
    ...
```

Calls generator to get different 

generator holds all the fake methods. first_name is not defined in address provider, but in names provider. Still this is fine! It will use reg expression to parse the the method. For example, if city_formats are 
('{{first_name}} {{city_suffix}}', ), then essentailly the generator would call
generator.first_name and generator.city_suffix,, and concantate the results together.

How does faker.first_name()?

Cross-referencing different fake methods from providers

Locale support
When a user specifies locales, faker identifies the locale-specific provider if possible, and use default otherwise.

```python
    @classmethod
    def _get_provider_class(cls, provider, locale=''):

        provider_class = cls._find_provider_class(provider, locale)

        if provider_class:
            return provider_class, locale

        if locale and locale != DEFAULT_LOCALE:
            # fallback to default locale
            provider_class = cls._find_provider_class(provider, DEFAULT_LOCALE)
            if provider_class:
                return provider_class, DEFAULT_LOCALE

        # fallback to no locale
        provider_class = cls._find_provider_class(provider)
        if provider_class:
            return provider_class, None

        msg = 'Unable to find provider "{0}" with locale "{1}"'.format(
            provider, locale)
        raise ValueError(msg)
```

## Comparison With Other Packages
Two similar open source projects we discussed before
Let's have a comparison:

fake user agents:  crawls the latest user agents from external webpages, with proper caching. Dedicated to generating the accurate user agents.

A predefined set of user agents string, limited coverage, but good enough for most use cases though.

locust: for reading the user defined configuration.
Also applies meta-programming for defining locust using pure python. Uses __import__ for loading user-defined Locusts.

What can we implement faker otherwise?

Put all the fake data in json format rather than binding them in Python code. To provide the same interface, 

The use of [proxy pattern](https://en.wikipedia.org/wiki/Proxy_pattern).
Without the setattr trick - define the methods in each provider.

To support direct access to each provider method, construct a dictionary and map the method user wants to call to the provider method.

Introspection a bit more difficult

Have to call some other provider methods and import them to circular import. A messy import graph maybe?

No use cases to support the config changes.

## Summary
As a quick recap, faker extensively uses meta-programming to build an extensible framework.
