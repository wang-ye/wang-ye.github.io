# Python Logging Tips

https://dzone.com/articles/best-practices-python-logging
Putting self.logger = logging.getLogger(type(self).__name__) on a base class is a good way to get a unique logger for each subclass, without each subclass having to set up their own logger.

Almost never use printing. Use logging, and set your logger(s) up to log to stdout with a StreamHandler while you are debugging. Then you can leave your ‘prints’ in, which will make life easier when you need to go back in to find bugs.

conditional logging

log emit
log routing



Why handlers?

https://fangpenlin.com/posts/2012/08/26/good-logging-practice-in-pythonq/

## Handlers
From [The Hitchhiker’s Guide to Python](http://docs.python-guide.org/en/latest/) -
It is strongly advised that you do not add any handlers other than NullHandler to your library’s loggers.


## How to Disable Logging For a Lib
Simple trick to disable lib warnings:

https://stackoverflow.com/questions/11029717/how-do-i-disable-log-messages-from-the-requests-library

