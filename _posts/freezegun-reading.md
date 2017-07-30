Freezegun

Mostly used in tests.

```python
from freezegun import freeze_time
import datetime
import unittest

# Use decorator.
@freeze_time("2012-01-14")
def decorator_test():
 assert datetime.datetime.now() == datetime.datetime(2012, 1, 14)

# Use context manager.
def context_manager_test():
    assert datetime.datetime.now() != datetime.datetime(2012, 1, 14)
    with freeze_time("2012-01-14"):
        assert datetime.datetime.now() == datetime.datetime(2012, 1, 14)
    assert datetime.datetime.now() != datetime.datetime(2012, 1, 14)
```

Now assume we want to support these two use cases. How to achieve?

Use 


## Misc:
The use of __all__ in __init__.py:
This is to support from package import *. Only