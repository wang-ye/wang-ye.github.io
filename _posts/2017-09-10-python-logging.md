---
layout: post
title:  "Python Logging Tips"
date:   2017-09-10 19:59:03 -0800
---
I want to summarize some Python logging basics and useful tips in this blog.

## No Printing
Use logging rather than *print* method. You can always redirect your logger to stdout via StreamHandler while debugging.

## Logging in Classes
From some [logging practices](https://dzone.com/articles/best-practices-python-logging), putting self.logger = logging.getLogger(type(self).__name__) on a base class is a good way to get a unique logger for each subclass, without each subclass having to set up their own logger.

## Routing Logs
In Python, handlers are used to route logs to different locations.

```python 
# create logger with 'spam_application'
logger = logging.getLogger('spam_application')
logger.setLevel(logging.DEBUG)
# create file handler which logs even debug messages
fh = logging.FileHandler('spam.log')
fh.setLevel(logging.DEBUG)
# create console handler with a higher log level
ch = logging.StreamHandler()
ch.setLevel(logging.ERROR)
# create formatter and add it to the handlers
formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
fh.setFormatter(formatter)
ch.setFormatter(formatter)
# add the handlers to the logger
logger.addHandler(fh)
logger.addHandler(ch)

logger.info('creating an instance of auxiliary_module.Auxiliary')
a = auxiliary_module.Auxiliary()
logger.info('created an instance of auxiliary_module.Auxiliary')
logger.info('calling auxiliary_module.Auxiliary.do_something')
a.do_something()
logger.info('finished auxiliary_module.Auxiliary.do_something')
logger.info('calling auxiliary_module.some_function()')
auxiliary_module.some_function()
logger.info('done with auxiliary_module.some_function()')
logger.error('an error happened')
```

Running the above code will generate a ``spam.log`` file, as well as a console output:
```shell
2017-09-20 00:05:51,667 - spam_application - ERROR - an error happened
```

Also, note that sometimes you need to set disable_existing_loggers to False. You can find the example [here](https://github.com/wang-ye/code/blob/master/daily/09_13_17/main2.py).

## Library Logging
In [The Hitchhiker’s Guide to Python](http://docs.python-guide.org/en/latest/) , It is strongly advised that you do not add any handlers other than NullHandler in your library’s loggers.

## Disabling Logging For a Lib
Here is a [simple trick](https://stackoverflow.com/questions/11029717/how-do-i-disable-log-messages-from-the-requests-library ) to disable lib warnings.
