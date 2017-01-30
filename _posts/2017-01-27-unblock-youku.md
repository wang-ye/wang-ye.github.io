---
layout: post
title:  "How Unblock Youku Works"
date:   2017-01-27 19:40:01 -0800
---

### Easily Accessing Chinese Website
On a lot of the Chinese online streaming website, such as Youku (Chinese version of Youtube), for copyright reasons, many oversea IPs will be blocked. Unblock Youku helps people to watch Chinease TV shows and many more.

### Structure

chrome

node server backend

Supports several modes

normal mode:
proxy server in China

#### mode setup

When user chooses 'normal' mode,

```
function setup_mode_settings(mode_name) {
    ...
    case 'normal':
        setup_header();
        setup_proxy();
        chrome.browserAction.setBadgeText({text: ''});
        change_browser_icon('normal');
    case ...
```

setup_header essentially assigns a random IP address for any given request when a user is accessing certain URLs:

```
chrome.webRequest.onBeforeSendHeaders.addListener(
    header_modifier,
    {
        // Only applicable when the URL is among unblock_youku.header_urls
        urls: unblock_youku.header_urls
    },
    ['requestHeaders', 'blocking']
);

// Insert the Client-IP and X-Forwarded-For header to fake the IP address.
function header_modifier(details) {
    console.log('modify headers of ' + details.url);

    details.requestHeaders.push({
        name: 'X-Forwarded-For',
        // unblock_youku.ip_addr is a fake IP served by China Telecom.
        value: unblock_youku.ip_addr
    }, {
        name: 'Client-IP',
        value: unblock_youku.ip_addr
    });

    return {requestHeaders: details.requestHeaders};
}
```

When a request arrives, if the extension detects that the request URL is among one of the IPs that just changing headers of it will be sufficient to bypass the blocking, then it adds the headers.

What is X-Forwarded-For and Client-IP?

How to set up the proxy?


The mapping files:

```
    'v.youku.com': [
      /^\/player\//i,
      /^\/v_show\//i
    ],
    'api.youku.com': [
      /^\/player\//i
    ],
    ...
```

FindProxyForURL is called.
regex matching, and then send it to proxy server
'HTTPS secure.uku.im:8443; HTTPS secure.uku.im:993; DIRECT;'

squid proxy in China

Some example URLs that are allowed by the squid proxy.
Details are in: https://github.com/uku/Unblock-Youku/wiki/%E5%9C%A8%E7%BE%8E%E5%9B%A2%E4%BA%91%E6%9E%B6%E8%AE%BE%E8%87%AA%E5%B7%B1%E7%9A%84-Unblock-Youku-%E4%BB%A3%E7%90%86%E6%9C%8D%E5%8A%A1%E5%99%A8
squid

```
^http:\/\/www\.tudou\.com\/.*$
^http:\/\/www\.tudou\.com\/.*$
^http:\/\/www\.tudou\.com\/.*$
^http:\/\/www\.tudou\.com\/.*$
^http:\/\/s\.plcloud\.music\.qq\.com\/.*$
^http:\/\/i\.y\.qq\.com\/.*$
^http:\/\/i\.y\.qq\.com\/.*$
^http:\/\/c\.y\.qq\.com\/.*$
^http:\/\/c\.y\.qq\.com\/.*$
...
```

In Squid script, the setup sets "forwarded_for transparent", which does not append anything to X-Forwarded-For. Also, it only allows URLs in the crx_url list and rejects other URLs.
Detailed link at : https://gist.githubusercontent.com/zhuzhuor/6b50406a9040e5c0b79d/raw/5e22f4b94158baacbb2c8b314c47e7ba763bbf6d/squid.conf

light mode:
no proxy needed.
Modify http headers + redirect the IP address checking request to Unblock-Youku's own proxy.
.

### Learnings
1. Often times we want to check the logs in the background page. To do so, enable developer mode. On the extensions you can see "inspect views: background" page.

2. chrome://net-internals/#proxy


### Open Questions
How is the background page used for here?
Why using storage? How is the storage used?
How is the extension started?
What are the URLs in the list? Can you map them to some real Youku urls?
How to access geo-blocked content? What is the general approach to do so?

VPN VS DNS change?

Code structure/orgnization? What new stuff did you learn?

Redirect mode?
