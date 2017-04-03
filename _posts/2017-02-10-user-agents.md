---
layout: post
title:  "Understanding Fake User Agents Generation"
date:   2017-02-10 19:59:03 -0800
---
For web developers, user agent concept is a must-know.
User agent serves as a middleman between the user and the Internet. When you browse a website, your browser identifies itself as a user agent, and sends the browser and device information to the server. On the server side, developers can improve the user experience by returning customized CSS or Javascript for the surfer. A user agent can also be crawlers like Google robot, or an offline browser such as wget. [This article](http://www.whoishostingthis.com/tools/user-agent/) provides a good introduction.

As a concrete example, my browser user agent is

```
Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko)
Chrome/55.0.2883.87 Safari/537.36
```

This user agent string contains information about OS, browser version, browser type, the rendering engine and other information. "Mozilla/5.0" here exists mostly for historical reasons and have no real meaning. "Linux x86_64" means my OS is a Linux 64bit. I am browsing with Chrome of version  55.0.2883.87.

Often times we want to use some fake user agents. This is sometimes called [user agent spoofing](http://www.whoishostingthis.com/tools/user-agent/). On the bright side, many browsers can identify themselves as other browsers, for compatibility or development purposes. The browser on Android devices is set to spoof Safari to avoid problems when loading content. On the dark side, hackers can use fake agents to send requests to web server or crawl website.

Fake user-agent is a Python tool to randomly generate user agents based on Internet browsing frequency. Inside settings.py, it specifies the web pages to crawl the different browsers with frequency stats and the user agents.

```Python
# Inside settings.py
BROWSERS_STATS_PAGE = 'http://www.w3schools.com/browsers/browsers_stats.asp'

BROWSER_BASE_PAGE = 'http://useragentstring.com/' +
    'pages/useragentstring.php?name={browser}'
```

The core logic lies in utils.py#load method. Basically, fake-useragents first crawls all the available browsers and their frequency on *BROWSERS_STATS_PAGE*, and then gets the user agents from *BROWSER_BASE_PAGE*.
when user queries for a certain browser version, it randomly pick a user agent from the crawled lists based on the frequency.

For better user experience, fake-useragents also supports two kinds of caching. 

```Python
# Inside settings.py
__version__ = '0.1.4'

# Storing generated fake agents locally in JSON.
DB = os.path.join(
    tempfile.gettempdir(),
    'fake_useragent_{version}.json'.format(version=__version__,),
)

CACHE_SERVER = 'https://fake-useragent.herokuapp.com/browsers/{version}'.
    format(version=__version__,)
```

1. User can store the fake agent list locally in JSON and later retrieves it directly. 
2. fake-useragents provides a cache server, which contains the latest fake agents. In case the direct crawling of browsers or agents list fails, users can still read the cache server and get the fake agents.

To make it more usable, fake-useragents implements __getitem__ and __getattr__ method inside **fake.py**. Users can thus call the following to get a fake agent string.

```Python
# Copied from README.
ua = UserAgent()

# Internally calls __getattr__ to read fake agents.
ua.chrome
# Mozilla/5.0 (Windows NT 6.1) AppleWebKit/537.2 (KHTML, like Gecko) Chrome/22.0.1216.0 Safari/537.2'

# Internally calls __getitem__ to read fake agents.
ua['google chrome']
# Mozilla/5.0 (X11; CrOS i686 2268.111.0) AppleWebKit/536.11 (KHTML, like Gecko) Chrome/20.0.1132.57 Safari/536.11

```

This is a pretty useful tool for generating fake agents!
