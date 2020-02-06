Python added [lru_cache](https://docs.python.org/3/library/functools.html#functools.lru_cache) to functools since Python 3.2. It supports LRU caching scheme, and can also easily downgrade to a simple caching storage.
For example, when implementing dynamic programming solutions with memorization, lru_cache can be very handy:

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

Here, we are not using its LRU property, but treat it as a boundless cache. The output for the above program is

```
CacheInfo(hits=48, misses=51, maxsize=None, currsize=51)
```

The above lru_cache use case is achieved with the following [wrapper](https://github.com/python/cpython/blob/78c7183f470b60a39ac2dd0ad1a94d49d1e0b062/Lib/functools.py#L786):

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
The actual LRU cache implementation is a bit more complex. As we know, to implement the LRU cache you will need both [a dictionary and a linked list](https://leetcode.com/problems/lru-cache/discuss/46121/lru-cache-implementation-in-python-w-explanation). Linked list node insertion and deletion could be time consuming. The implementation, however, uses an optimized [circular linked list](https://github.com/python/cpython/blob/54b4f14712b9350f11c983f1c8ac47a3716958a7/Lib/functools.py#L773) so you never need to delete any nodes. Also, it uses a lock for Threadsafe control. Interested readers can refer to the source code for more details.