Understanding Flask

With a simple flask example, and explain how the code works.

```
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, World!'

if __name__ == '__main__':
    app.run()
```

## Startup of a Web Server
wsgi - web server gateway interface

web server

Actual App: What method to implement?



## Serving Request

client  - send requests
wsgi server - parse request, send request to application, get the response and return it back to client.
application and middleware - handles the request, generate response.


What is app?
app is part of the application. Where did we start the server?
Flask used a built-in server called Werkzeug. It is not a full blown webserver like Apache but a simple standalone one. When you call app.run(), server starts.

```

```

Now, let's say a request comes,

The WSGI server will parse the request, and call app.wsgi_app() method to process the request.

Defines multiple hooks. All happening inside full_dispatch_request(self) method.

routing

request

response and template
