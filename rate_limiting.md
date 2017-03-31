API Rate Limiting 

Rate limiting is used to control the traffic rate sent or received by a hardware or API. In this article we will focus on the API part. Most APIs, such as Google Maps, Twitter, Github, Slack all maintain a rate limiting scheme.
These APIs are subject to a limit on how many calls can be made per second (or minute, or other short time period), in order to protect servers from being overloaded and maintain high quality of service to many clients. This article dives into the implmentation of the rate limiter in Python.

For some concrete examples, 
To start with a simplified version of rate limiter:
http://flask.pocoo.org/snippets/70/

To design and implement a rate limiter, you may want to consider the following:

1. What do you plan to limit?
ip: The IP can send at most some requests.
user: Each user can at most make some amount of requests.
service: per endpoint

2. How do you define the limit?
You may want to have a simple parser to parse the rate limiting rules.
This makes the system more confiugrable.

3. How to store the rate limit information?
Need to keep counters about the number processed requests in certain time period.
store the existing requests, and the remaining request quota. persist?
It can be:
In memory,
redis
...

Deep dive into Python rate-limitter package
Use decorators to add the limit to a global config.
Use before_request to inject the check.
Use another package called limits to parse the limit and persist optionally.
Use after_request to update the headers. It reads the view limit in 'g', and attach proper headers to the response.

nginx has built-in rate limiting tools.