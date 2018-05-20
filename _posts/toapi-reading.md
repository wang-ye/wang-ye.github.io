
Why pass the cache_class in setting?

toapi is a powerful tool to provide APIs for any website.

To use, define the objects you want to retrieve using the Setting class. Here is a config example to create two APIs for ycombinator:

```
class MySettings(Settings):
    web = {
        "with_ajax": True,
        "request_config": {},
        "headers": None
    }


api = Api('https://news.ycombinator.com', settings=MySettings)


class Post(Item):
    url = XPath('//a[@class="storylink"]/@href')
    title = XPath('//a[@class="storylink"]/text()')

    class Meta:
        source = XPath('//tr[@class="athing"]')
        route = {'/news?p=:page': '/news?p=:page'}


class Page(Item):
    next_page = XPath('//a[@class="morelink"]/@href')

    class Meta:
        source = None
        route = {'/news?p=:page': '/news?p=:page'}

    def clean_next_page(self, next_page):
        return "http://127.0.0.1:5000/" + next_page


api.register(Page)
api.register(Post)

api.serve()
```

When accessing `http://127.0.0.1:5000/`, you can get the following:

```json
{
  "post": [
    {
      "title": "IPvlan overlay-free Kubernetes Networking in AWS", 
      "url": "https://eng.lyft.com/announcing-cni-ipvlan-vpc-k8s-ipvlan-overlay-free-kubernetes-networking-in-aws-95191201476e"
    }, 
    {
      "title": "Apple is sharing your facial wireframe with apps", 
      "url": "https://www.washingtonpost.com/news/the-switch/wp/2017/11/30/apple-is-sharing-your-face-with-apps-thats-a-new-privacy-worry/"
    }, 
    {
      "title": "Motel Living and Slowly Dying", 
      "url": "https://lareviewofbooks.org/article/motel-living-and-slowly-dying/#!"
    }
  ]
}
```

I want to discuss:
0. The diagram
1. The local server running the toapi app.
2. The scraping process - how to map the setting/entities to scrape criteria
3. Cache and storage
4. The metaclass use case

```python
class Server:
    def __init__(self, api, settings):
        app = Flask(__name__)
        app.logger.setLevel(logging.ERROR)
        self.app = app
        self.api = api
        self.settings = settings
        self.init_route()

    def init_route(self):
        app = self.app
        api = self.api

        @app.route('/')
        def index():
            base_url = "{}://{}".format(request.scheme, request.host)
            basic_info = {
                "items": "{}/_{}".format(base_url, "items"),
                "status": "{}/_{}".format(base_url, "status")
            }
            return jsonify(basic_info)

        @app.route('/_status')
        def status():
            status = {
                'cache_set': api.get_status('_status_cache_set'),
                'cache_get': api.get_status('_status_cache_get'),
                'storage_set': api.get_status('_status_storage_set'),
                'storage_get': api.get_status('_status_storage_get'),
                'sent': api.get_status('_status_sent'),
                'received': api.get_status('_status_received')
            }
            return jsonify(status)

        @app.route('/_items')
        def items():

            results = {}
            for alias, item in api.items.items():
                results[alias] = [i['item'].__name__ for i in item]
            return jsonify(results)

        @app.errorhandler(404)
        def page_not_found(error):
            start_time = time()
            path = request.full_path
            if path.endswith('?'):
                path = path[:-1]
            try:
                res = api.get_cache(path)
                if res is None:
                    res = api.parse(path)
                    api.set_cache(path, res)
                if res is None:
                    logger.error('Received', '%s 404' % request.url)
                    return 'Not Found', 404
                api.update_status('_status_received')
                end_time = time()
                time_usage = end_time - start_time
                logger.info(Fore.GREEN, 'Received',
                            '%s %s 200 %.2fms' % (request.url, len(res), time_usage * 1000))

                return app.response_class(
                    response=res,
                    status=200,
                    mimetype='application/json'
                )
            except Exception as e:
                return str(e), 500

    def run(self, ip='127.0.0.1', port=5000, **options):
        """Runs the application"""

        for _signal in [SIGINT, SIGTERM]:
            signal(_signal, self.stop)

        self.app.run(ip, port, **options)

    def stop(self, signal, frame):
        logger.info(Fore.WHITE, 'Server', 'Server Stopped')
        exit()
```

Web server