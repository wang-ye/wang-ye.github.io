WSGI Flask

I am going to write a Flask series to understand the internals of Flask and its ecosystem. This post assumes basic knowledge about Flask.

The server startup
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
Common Gateway Interface - a mechnism to communicate between the web server and the application through passing the environment variables.

wsgi - web server gateway interface


One reason is that it allows you to write an application as a 
generator.  But more importantly, it's necessary in order to support 
'write()' for backward compatibility with existing frameworks, and that's 
pretty much the "killer reason" it's structured how it is.  This particular 
innovation was Tony Lownds' brainchild, though, not mine.  In my original 
WSGI concept, the application received an output stream and just wrote 
headers and everything to it.

Web server can be Apache, ngiux or Gunicorn.

For Flask case, which web server is invoked? How does the web server do? or communicate with Flask app?

web server

Actual App: What method to implement?

app can be nested.

What's the call sequence?

Flask performance

Faskcgi / cgi