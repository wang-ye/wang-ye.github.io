Understanding User agents
How does fake user agents work?

two modes - cached VS realtime crawling

It is worth noting that some websites, such as http://useragentstring.com
has all the user agents information available.
In a nutshell, fake-useragents just crawls all the available agents from the web, and, when user queries for a certain browser version, it randomly pick a user agent from the crawled list.

Another mode is caching - https://fake-useragent.herokuapp.com/browsers/AGENT_VERSION contains a cached version of the user agents. Once the realtime crawling is no longer working, the tool can read the agents from the above site.

Finally, the user can also keep a local copy of the agents in json format.

Something good to know:
1. tox - It is a tool for automatic testing locally.
2. travis to set up CI, and
3. coverage to generate coverage report.