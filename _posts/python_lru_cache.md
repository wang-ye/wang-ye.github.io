Python added lru_cache to functools since Python 3.2.

This post briefly examines its implementation and how we can use it.

For me, its biggest use case would be implementing dynamic programming solutions with memorization. For example, you can use

```python
import functools
@functools.lru_cache(maxsize=None)
def dp(val):
    if val in [0, 1]:
        return 1
    return dp(val - 1) + dp(val - 2)

dp(50)
print(dp.cache_info())
```

Here, we are not using its LRU property, but treat it as a boundless cache. The output is
```
CacheInfo(hits=48, misses=51, maxsize=None, currsize=51)
```

Its core implementation is also straightforward. Essentially it is the following [wrapper](https://github.com/python/cpython/blob/78c7183f470b60a39ac2dd0ad1a94d49d1e0b062/Lib/functools.py#L786):

```python
cache = {}
sentinel = object()          # unique object used to signal cache misses

def wrapper(*args, **kwds):
    # Simple caching without ordering or size limit
    nonlocal hits, misses
    key = make_key(args, kwds)  # Simplified key making.
    result = cache.get(key, sentinel)
    if result is not sentinel:
        hits += 1
        return result
    misses += 1
    result = user_function(*args, **kwds)
    cache[key] = result
    return result
```

# The LRU Impl
Threadsafe LRU cache.
Two interesting things to note. It uses a circular linked list so you never
need to delete any nodes in the linked list. Also, it uses a lock
for Threadsafe control.
The other