---
layout: post
title:  "Maintaining Free ProxyPool For Web Crawling"
date:   2017-09-01 19:59:03 -0800
---

Proxies are often used when crawling the website. In this post, I will discuss [ProxyPool](https://github.com/jhao104/proxy_pool), a Git repository that maintains a *free* proxy pool for crawling Chinese websites. It provides a simple Flask interface, so spiders can get the usable proxies by accessing *localhost:5000*.

```python
import requests

def get_proxy():
    # Return a random proxy extracted from the proxypool.
    return requests.get("http://127.0.0.1:5000/get/").content

# your spider code
def spider():
    # ....
    requests.get('https://www.example.com', 
        proxies={"http": "http://{}".format(get_proxy())})
    # ....
```

We will first talk about its high-level design. Next, we dive into some implementation details such as proxy discovery and verification.

# ProxyPool Design
The Github already has the proxyPool diagram.

![]({{ site.url }}/assets/proxy-pool.png)

The intended users of the proxy pool is the *spiders (crawlers)*. They talk directly ProxyPool API and gets the available proxies. ProxyPool runs several background jobs to schedule the proxy crawling from several websites providing free proxies.

It contains several components:

* API: end users (crawlers) talks with ProxyPool through this API.
* DB: Store the usable proxies. It provides APIs to read/delete/write proxies.
* Schedulers: Run multiple threads together to retrieve, update and validate proxies. Also, it deletes unusable ones.
* ProxyGetter: The library to retrieve proxies from free websites.

The design is clean and straightforward. As to the actual implementation, it is worth pointing out a couple of interesting pieces.

## Implementation Decisions
In this section,, we will discuss key implementation decisions.

## Flask App
The App provides APIs to get usable proxies and remove stable proxy. I will skip this part and interested readers can check the [source](https://github.com/jhao104/proxy_pool/blob/master/Api/ProxyApi.py) directly.

# Background Job Scheduling
ProxyPool schedules periodical jobs such as proxy crawling and removing stale proxies. Instead of relying a heavyweight celery scheduling approach, it uses a multi-processes approach.

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

Among the three processes, *ProxyApiRun* supports the Flask app to interact with crawlers; *RefreshRun* crawls the websites containing free proxies and store newly-emerged proxies locally in DB, while *ValidRun* validates the proxies with the following approach:

```python
def validUsefulProxy(proxy):
    # proxy var is the proxy we want to verify.
    proxies = {"https": "https://{proxy}".format(proxy=proxy)}

    r = requests.get(
            'https://www.baidu.com', proxies=proxies, timeout=40, verify=False)
    if r.status_code == 200:
        logger.debug('%s is ok' % proxy)
        return True
    return False
```

## Choosing the Right DB
ProxyPool supports different DB backends such as Mongodb, Redis and [SSDB](https://github.com/ideawu/ssdb). User dictates which DB to use in configuration. ProxyPool utilizes the factory pattern for choosing the right DB.

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

# metaclass here uses Py3 syntax.
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
        self.client = getattr(__import__(__type), __type)(
            name=self.config.db_name,
            host=self.config.db_host,
            port=self.config.db_port
        )
    ...
```

DBClient is a client factory. It maintains a *client*, and it is set based on config. Also, it is implemented as a singleton with the help of metaclass.
Why making it a Singleton? DBClient caches the connection and potentially minimizes the number of connections to DB.

A different way of factory pattern: https://github.com/gennad/Design-Patterns-in-Python/blob/master/factory.py

## Get Proxies
ProxyPool gets the free proxies by crawling the websites that provide free proxies. Users can specify which websites to crawl in the config. In config, you will find something like this:

```ini
[ProxyGetter]
;register the proxy getter function
freeProxyFirst  = 1
freeProxySecond = 1
freeProxyThird  = 1
freeProxyFourth = 1
freeProxyFifth  = 1
```

Each of the *freeXXX* is defined as a static method in GetFreeProxy class:

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

Actual usage happens like this:

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
                    proxy_set.add(proxy.strip())

            # store raw proxy
            self.db.changeTable(self.raw_proxy_queue)
            for proxy in proxy_set:
                self.db.put(proxy)
```

The code applies *getattr* method to dynamically extract the website to crawl.
More specifically, the statement ``getattr(GetFreeProxy, proxyGetter.strip())()`` actually calls GetFreeProxy's methods as defined in *ini* file.

### Proxy Refresh and Validation 
Free proxies expire very fast. Thus, ProxyPool also provides some mechanism to detect new proxies and remove stale ones.

For identifying new proxies:

## Testing
All tests lie in *Test* directory. One final note about how to run the tests. Directly running the test files will cause import errors:

```shell
$ python -m Test/testLogHandler.py
Traceback (most recent call last):
  File "Test/testLogHandler.py", line 15, in <module>
    from Util.LogHandler import LogHandler
ImportError: No module named Util.LogHandler
```

Running it with `-m` will make sure everything is working:

```shell
$ python -m Test.testLogHandler
```

