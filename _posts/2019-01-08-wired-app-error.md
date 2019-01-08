I was running two hobby projects under the same server. It turned out both of them shared the same port, and thus one service was not getting requests as expected. To combat the issue, I changed that service port number to *6000*.

Wired things happened immediately - I could not access the new web service in Chrome! Here is my debug journey ...

## First Attempt - Keep only One Service Running
My cloud server is running two services at the same time. Could this be the issue? Can the server blocking two services running at the same time? To test this hypothesis, I turned down the other service, leaving only the one with port 6000 running. Unfortunately, this did not resolve my issue.

*Conclusion*: This is not the root cause.

## Second Attempt - Anything Wrong with Port Mapping?
To provide more context, I used docker-compose to start my service in a container (a58526758a4c), and also set the port mapping there. Could it be that my port mapping is not working as expected?
Docker has a command called `docker port` to inspect the port mapping.

```shell
docker port a58526758a4c
6000/tcp -> 0.0.0.0:6000
```
The mapping looks correct to me. For those who does not use docker, it basically says the container maps its 6000 port to the server's 6000 port, and 0.0.0.0 makes the server publicly visible. I then went to both the container and server, trying to run curl on *localhost:6000*. Surprisingly, both returned the valid result.

*Conclusion: Port mapping is also correct.*

## Final Attempt - Browser Issue?
I was really lost and could not figure out the issue. How about running curl on my desktop machine and report this as an issue? I then ran the same curl command on my desktop. Even more surprisingly, I can still get the valid response! So now this looks like a browser issue.

Why can't I visit it directly from Chrome? I tried Incongnito mode but still with no luck. I then changed to Firefox, this time I got some valuable hints.

```shell
This address is restricted

This address uses a network port which is normally used for purposes other than Web browsing. Firefox has canceled the request for your protection.
```

OK! Seems I am using some unsafe port, and both Chrome and Firefox is blocking me. This [stackoverflow post](https://superuser.com/questions/188058/which-ports-are-considered-unsafe-on-chrome) explains it well - *6000* is the default port of X11.

## The Takeaway
Check the unsafe port mapping when you choose a port. Or choose a port with a large value (>= 10000). Otherwise, you will waste hours just like me.