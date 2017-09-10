---
layout: post
title:  "Maintaining Free ProxyPool For Web Crawling"
date:   2017-09-01 19:59:03 -0800
---

To crawl the website effectively, proxies are often used. Today, I will discuss a [github repository](https://github.com/jhao104/proxy_pool) that maintains a *free* proxy pool for crawling purposes. Here is an example use of the proxy pool, assuming that a ProxyPool Flask App is running on localhost:5000.

```python
import requests

def get_proxy():
    # Return a random proxy extracted from the proxypool.
    return requests.get("http://127.0.0.1:5000/get/").content

# your spider code
def spider():
    # ....
    requests.get('https://www.example.com', proxies={"http": "http://{}".format(get_proxy())})
    # ....
```

# Proxy Pool Design
Here is the diagram for proxyPool design.

(https://camo.githubusercontent.com/583fcc7cd62289814f86f52a0152835b11802979/68747470733a2f2f706963322e7a68696d672e636f6d2f76322d66323735366461323938366161386138636162316639353632613131356235355f622e706e67)

It contains several components:

* API: end users (crawlers) talks with ProxyPool through this API.
* DB: Store the usable proxies. It provides APIs to read/delete/write proxies.
* Schedulers: Run multiple threads together to retrieve, update and validate proxies. Also, it deletes unusable ones.
* ProxyGetter: The library to retrieve proxies from free websites.

The design is a clean and straightforward. The code is also easy to follow, but it is worth pointing out a couple of interesting pieces:

# Background Job Scheduling
Instead of having heavyweight celery jobs for scheduling crawlers and removing stale proxies, it uses a multi-process approach.

```python
# The main.py
from Api.ProxyApi import run as ProxyApiRun
from Schedule.ProxyValidSchedule import run as ValidRun
from Schedule.ProxyRefreshSchedule import run as RefreshRun

def run():
    p_list = [
        Process(target=ProxyApiRun, name='ProxyApiRun'),
        Process(target=ValidRun, name='ValidRun'),
        Process(target=RefreshRun, name='RefreshRun')
    ]
    for p in p_list:
        p.start()
    for p in p_list:
        p.join()
```

Among the three processes, *ProxyApiRun* supports the Flask app to interact with crawlers; *RefreshRun* crawls the websites containing free proxies and store them locally, while *ValidRun* validates the proxies by applying them for the following crawling task:

```python
# proxy var is the proxy we want to verify.
proxies = {"https": "https://{proxy}".format(proxy=proxy)}

r = requests.get(
        'https://www.baidu.com', proxies=proxies, timeout=40, verify=False)
if r.status_code == 200:
    logger.debug('%s is ok' % proxy)
    ..
```

## Flask App
The app provides melthods to get usable proxies and remove a proxy if it is no longer available. Interested readers can check the source directly.

## The Factory Pattern for Choosing DB
ProxyPool provides different DB clients for Mongodb, Redis and [SSDB](). User dictates which DB to use in configuration.

```python
class Singleton(type):
    """
    Singleton Metaclass
    """

    _inst = {}

    # metaclass intercepts calls to instance creation.
    def __call__(cls, *args, **kwargs):
        if cls not in cls._inst:
            cls._inst[cls] = super(Singleton, cls).__call__(*args)
        return cls._inst[cls]

class DBClient(object, metaclass=Singleton):
    def __init__(self):
        self.config = GetConfig()
        self.__initDbClient()

    def __initDbClient(self):
        __type = None
        if "SSDB" == self.config.db_type:
            __type = "SsdbClient"
        elif "REDIS" == self.config.db_type:
            __type = "RedisClient"
        else:
            pass
        assert __type, 'type error, Not support DB type: {}'.format(self.config.db_type)
        self.client = getattr(__import__(__type), __type)(name=self.config.db_name,
                                                          host=self.config.db_host,
                                                          port=self.config.db_port)
    ...
```

DBClient is a client factory. It maintains a *client*, and it is set based on config. Also, it is implemented as a singleton with the help of metaclass.
Why making it a Singleton? DBClient caches the connection and potentially minimizes the number of connections to DB.

## Get Proxies
The basic idea is crawling the websites that provide free proxies. Users can specify which websites to crawl in the config.

```python
class GetFreeProxy(object):
    @staticmethod
    @robustCrawl  # decoration print error if exception happen
    def freeProxyFirst(page=10):
        url_list = ['http://www.data5u.com/',
                    'http://www.data5u.com/free/',
                    'http://www.data5u.com/free/gngn/index.shtml',
                    'http://www.data5u.com/free/gnpt/index.shtml']
        for url in url_list:
            html_tree = getHtmlTree(url)
            ul_list = html_tree.xpath('//ul[@class="l2"]')
            for ul in ul_list:
                yield ':'.join(ul.xpath('.//li/text()')[0:2])

    @staticmethod
    @robustCrawl
    def freeProxySecond(proxy_number=100):
        url = "http://www.66ip.cn/mo.php?sxb=&tqsl={}&port=&export=&ktip=&sxa=&submit=%CC%E1++%C8%A1&textarea=".format(
                proxy_number)
        request = WebRequest()
        # html = request.get(url).content
        # content为未解码，text为解码后的字符串
        html = request.get(url).text
        for proxy in re.findall(r'\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}:\d{1,5}', html):
            yield proxy
```

It defines different methods to crawl different proxy sites. In config, register it as

```ini
[ProxyGetter]
;register the proxy getter function
freeProxyFirst  = 1
freeProxySecond = 1
freeProxyThird  = 1
freeProxyFourth = 1
freeProxyFifth  = 1
```

Actual crawling happens like this:

```python
class ProxyManager(object):
    ...
    def refresh(self):
        """
        fetch proxy into Db by ProxyGetter
        """
        # Reads the functions in config.
        for proxyGetter in self.config.proxy_getter_functions:
            proxy_set = set()
            # fetch raw proxy
            for proxy in getattr(GetFreeProxy, proxyGetter.strip())():
                if proxy.strip():
                    self.log.info('{func}: fetch proxy {proxy}'.format(func=proxyGetter, proxy=proxy))
                    proxy_set.add(proxy.strip())

            # store raw proxy
            self.db.changeTable(self.raw_proxy_queue)
            for proxy in proxy_set:
                self.db.put(proxy)
```

## Scheduler
Multi-threading scheduler with apscheduler
### Refresh
Every few minutes, a scheduled job goes through
### Validate
Gets 


## Testing
All tests lie in *Test* directory. One final note about how to run the tests.

```shell
$ python -m Test.testLogHandler
```

Directly running the test files will casue import errors.

```shell
$ python -m Test/testLogHandler.py
Traceback (most recent call last):
  File "Test/testLogHandler.py", line 15, in <module>
    from Util.LogHandler import LogHandler
ImportError: No module named Util.LogHandler
```

## Takeaway
What can we learn from this design?

A use case for factory and singleton pattern.