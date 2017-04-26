---
layout: post
title:  "Rate Limiting Part One - A Flask Example"
date:   2017-04-23 19:59:03 -0800
---

In this post I will discuss the fundamentals of rate limiting. Rate limiting is used to control the traffic rate sent or received by a hardware or software APIs. We will focus on API rate limiting.

Most APIs, such as Google Maps, Twitter and Stripe have rate limiting mechanisms. APIs are subject to a limit on how many calls can be made per second (or minute, or other short time period), in order to protect servers from being overloaded. [Stripe](https://stripe.com/blog/rate-limiters) recently published a very good overview about its rate limiters. 

To design and implement an API rate limiter, you may want to consider the following:

**What to limit?**
Different kinds of API rate limiting exist.
Basically, **in a certain time span**, rate limiting can be

1. ip-based: The API serves at most X requests for a given IP.
2. user-based: The API serves at most X requests for a user.
3. concurrency-based: The API serves at most X *concurrent* requests for a user or IP.
4. Combination of the above: For example, An API serves at most X requests per IP per user.

**How to define limit?**
To represent the limits like "allow at most 3 requests per second", one can hard-code it in sources, or define a rule format and build a simple parser to parse the rules.

**Where to store request statistics?**
For effective rate limiting, the limiter must store statistics like number of requests served in the past X minutes. The storage can be memory, key-value stores such as Redis and Memcached, and even SQL databases.

**How to notify end user about reaching rate limit?**
The users hitting rate limit often trigger the requests with scripts, and the API should let the user know *why* they are limited. This is often achieved via response headers. One example from Twitter is [here](https://dev.twitter.com/rest/public/rate-limiting).

Now, imagine that you want to have a quick rate limiting hack on top of Flask, and you have the following requirements:

1. For a given API endpoint, it restricts each user to X requests per second.
2. Redis-backed storage. Redis is thread safe, and it has been used for other tasks, such as providing global locking. Here we use it to store the seen requests.
3. We will hard-code the limits in the source code for simplicity.
4. We will add headers like X-RateLimit-Limit and X-RateLimit-Reset to notify the end users about the rate limit.

what would you do? Fortunately, we already had an [implementation](http://flask.pocoo.org/snippets/70/) here, written by well-known open source contributor [Armin Ronacher](http://lucumr.pocoo.org/about/), who is also the creator of Flask.

![Flask Request Flow]({{ site.url }}/assets/flask-request-flow.png)

The above diagram shows the basic Flask request flow. To implement rate limiting, We do the following. First, right before Flask executes the endpoint, it uses Redis pipeline to update the request statistics. If the current request has reached the rate limit, the code returns the rate limit error message. This all happens inside a decorator called ratelimit. BTW, we can also bookkeep each request in before_request hook. Second, right before the end of the request cycle, the code adds the rate-limit headers to the response object in the after_request callback. Again, the detailed code can be found [here](http://flask.pocoo.org/snippets/70/). I pasted decorator for API endpoint for your reference.

```python
def ratelimit(limit, per=300, send_x_headers=True,
              over_limit=on_over_limit,
              scope_func=lambda: request.remote_addr,
              key_func=lambda: request.endpoint):
    def decorator(f):
        def rate_limited(*args, **kwargs):
            key = 'rate-limit/%s/%s/' % (key_func(), scope_func())
            rlimit = RateLimit(key, limit, per, send_x_headers)
            g._view_rate_limit = rlimit
            if over_limit is not None and rlimit.over_limit:
                return over_limit(rlimit)
            return f(*args, **kwargs)
        return update_wrapper(rate_limited, f)
    return decorator
```

## Summary
This blog discusses the fundamentals of API rate limiting, and illustrated a simple rate limiting example on top of Flask. In the next part of the rate limiting series, we will look at the Flask rate limiting extension, as well as a ruby implementation.
