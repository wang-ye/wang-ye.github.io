[toapi](https://github.com/gaojiuli/toapi) is a tool to provide APIs for any website. It is very small - the key functionality has less than 500 lines. However, it contains several nice features worth learning. In this post I want to discuss how to design and implement such a tool.

## The Problem
For websites without direct API access, build a tool to provide a principled way to extract data from them. Users only needs to specify the web pages and the fields to extract the data from. To design such a tool, we need to have:

1. the URL(s) to extract the data from
2. the fields we want to access 
3. a running web server handling API requests

To better understand the problem here, the *toapi* repo provides an [example](https://github.com/gaojiuli/toapi/blob/master/examples/hackernews_page.py) by providing [Hackernews](http://news.ycombinator.com/) an API! All you need to make it work is having the following config-like definition:

```
from flask import request
from htmlparsing import Attr, Text

from toapi import Api, Item

api = Api(browser='geckodriver')

@api.site('https://news.ycombinator.com')
@api.list('.athing')
@api.route('/posts?page={page}', '/news?p={page}')
@api.route('/posts', '/news?p=1')
class Post(Item):
    url = Attr('.storylink', 'href')
    title = Text('.storylink')


@api.site('https://news.ycombinator.com')
@api.route('/posts?page={page}', '/news?p={page}')
@api.route('/posts', '/news?p=1')
class Page(Item):
    next_page = Attr('.morelink', 'href')

    def clean_next_page(self, value):
        return api.convert_string('/' + value, '/news?p={page}', request.host_url.strip('/') + '/posts?page={page}')


api.run(debug=True, host='0.0.0.0', port=5000)
```

Here we define two classes, *Post* and *Page*, all inheriting from *Item* class. We will discuss the *Item* class later but for now just think it as the base class to define the fields we care. The `site` decorator defines the base url to crawl the data from, while the `route` details the remaining paths. In the two Item-inherited classes, we specify the fields to crawl. More specifically,

```python
url = Attr('.storylink', 'href')
```

means to extract the *'href'* attribute in *'.storylink'* element, and the extracted data will be renamed as *'url'* in our JSON response. Finally, the *list* decorator tells there are multiple urls in each given page.

When accessing `http://127.0.0.1:5000/`, you can get response like:

```json
{
  "Page": {
    "next_page": "http://127.0.0.1:5000/posts?page=2"
  }, 
  "Post": [
    {
      "title": "Mathematicians Crack the Cursed Curve", 
      "url": "https://www.quantamagazine.org/mathematicians-crack-the-cursed-curve-20171207/"
    }, 
    {
      "title": "Stuffing a Tesla Drivetrain into a 1981 Honda Accord", 
      "url": "https://jalopnik.com/this-glorious-madman-stuffed-a-p85-tesla-drivetrain-int-1823461909"
    }
  ]
}
```

Next, we will see how *toapi* reads the configurations above and runs the API web server.

## Metaclass Configuration
*Toapi* uses metaclass to elegantly reads the configurations. The *Item* class, which dictates what fields to extract for the web pages, is in fact a metaclass:

```python
class ItemType(type):
    def __new__(cls, what, bases=None, attrdict=None):
        __fields__ = OrderedDict()

        for name, selector in attrdict.items():
            if isinstance(selector, Selector):
                __fields__[name] = selector

        for name in __fields__.keys():
            del attrdict[name]

        instance = type.__new__(cls, what, bases, attrdict)
        instance._list = None
        instance._site = None
        instance._selector = None
        instance.__fields__ = __fields__
        return instance


class Item(metaclass=ItemType):

    @classmethod
    def parse(cls, html: str):
        if cls._list:
            result = HTMLParsing(html).list(cls._selector, cls.__fields__)
            result = [cls._clean(item) for item in result]
        else:
            result = HTMLParsing(html).detail(cls.__fields__)
            result = cls._clean(result)
        return result

    @classmethod
    def _clean(cls, item):
        for name, selector in cls.__fields__.items():
            clean_method = getattr(cls, 'clean_%s' % name, None)
            if clean_method is not None:
                item[name] = clean_method(cls, item[name])
        return item
```

Here the ItemType class derives from `type` directly. It converts all the class fields with a selector value into ```__fields__``` dict. This affects all subclasses as well. For example, here is the *Page* class:

```python
@api.site('https://news.ycombinator.com')
@api.route('/posts?page={page}', '/news?p={page}')
@api.route('/posts', '/news?p=1')
class Page(Item):
    next_page = Attr('.morelink', 'href')

    def clean_next_page(self, value):
        return api.convert_string('/' + value, '/news?p={page}', request.host_url.strip('/') + '/posts?page={page}')
```

Eventually, we will see **Page.__fields__** containing ('next_page', Attr('.moreline', 'href')). This is a clean way to provide configuration data, similar to what we did in [Django models](https://docs.djangoproject.com/en/2.0/topics/db/models/). The Item class also defines the *parse* and *_clean* methods. They conduct the actual heavylifting by extracting the data from raw HTML data. The *_clean* method does necessary post-processing on the fields.

## Flask Server
Next, we need to start a local server to serve the different API requests. This is achieved by utilizing a [Flask](http://flask.pocoo.org/) server.

```python
@self.app.route('/<path:path>')
def handler(path):
    try:
        start_time = time()
        full_path = request.full_path.strip('?')
        results = self.parse_url(full_path)
        end_time = time()
        time_usage = end_time - start_time
        res = jsonify(results)
        logger.info(Fore.GREEN, 'Received',
                    '%s %s 200 %.2fms' % (request.url, len(res.response), time_usage * 1000))
        return res
    except Exception as e:
        logger.error('Serving', f'{e}')
        logger.error('Serving', '%s' % str(traceback.format_exc()))
        return jsonify({'msg': 'System Error', 'code': -1}), 500


def parse_url(self, full_path: str) -> dict:
    results = self._cache.get(full_path)
    if results is not None:
        logger.info(Fore.YELLOW, 'Cache', f'Get<{full_path}>')
        return results

    results = {}
    for source_format, target_format, item in self._routes:
        parsed_path = self.convert_string(full_path, source_format, target_format)
        if parsed_path is not None:
            full_url = self.absolute_url(item._site, parsed_path)
            html = self.fetch(full_url)
            result = item.parse(html)
            logger.info(Fore.CYAN, 'Parsed', f'Item<{item.__name__}[{len(result)}]>')
            results.update({item.__name__: result})

    self._cache[full_path] = results
    logger.info(Fore.YELLOW, 'Cache', f'Set<{full_path}>')

    return results
```

The web server part is straightforward. It starts a server by serving all the *localhost:5000/\<path\>* requests. Multiple target paths can be matched to the path, and thus the *for* loop in the *parse_url* method. The results are converted in json format by calling `jsonify`.

## Summary
In this post, we discussed the design of the toapi tool. It uses metaclass to elegantly define the elements to extract from target websites, and start a local server to handle all requests. One thing to note is this is a lightweight tool without features to bypass user logins or other complex crawling.