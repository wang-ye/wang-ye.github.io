---
layout: post
title: "Unblock-Youku: Accessing Geo-blocked Website"
date:   2017-01-27 19:40:01 -0800
---

On a lot of the online streaming websites, such as Youku (Chinese version of Youtube), many non-Chinese IPs will be blocked. Unblock-Youku, a [Chrome extension](https://chrome.google.com/webstore/detail/unblock-youku/pdnfnkhpgegpcingjbfihlkjeighnddk?hl=en), helps people in other countries enjoy Chinese TV shows and many more. You can find its source code (mostly Javascript/Node.js) in [Github repo](https://github.com/uku/Unblock-Youku). In this blog, we will explore the underlying secrets of this powerful extension and learn some useful stuff regarding accessing geo-blocked websites using proxies.

## Structure
Unblock-Youku github repo has the following pieces:

1. chrome extension source code. The extension supports three modes. off, light and normal. "Off" disables all Unblock Youku functionalities. "Lite"    applies to only few websites, but can work with other proxy settings or software. The normal is the recommended one. It is fast, and applicable to most websites.
2. A redirection server backend written in node.js. This is how Unblock-Youku configures its redirection servers.
3. Configuration regarding the proxy server. The proxy server is built with Squid.

In this blog, I will focus on the normal mode, including the mode setup, request processing and the proxy server configuration. In the future I may cover the Lite mode in a separate blog. 

## Normal Mode Setup
How does Geo-based blocking work? One way to block Geo-restricted content is via URL addresses. A blocking service takes the input request, find the related fields and extract the IP address of the requester. The blocking service can take actions based on the region of the requester IP,

To combat this, Unblock-Youku identifies the URLs that are sent to the blocking service, and forges the IPs of these requests to trick the blocking service.

When user chooses 'Normal' mode, the Chrome extension marks the normal configuration in its storage, and setup the headers rewrite and proxy servers.

The initialization code lies in config.js. With a listener for "DOMContentLoaded" event, the code does the following:

```Javascript
// In config.js
document.addEventListener("DOMContentLoaded", function() {
    // Get the mode name,, if mode name is undefined, set it to Normal.
    get_mode_name(function(current_mode_name) {
        setup_mode_settings(current_mode_name);
        ...
    });
});
```

So by default, the extension works under "Normal" Mode. User can change the mode through the UI, which then calls set_mode_name to easily update the current mode. The get_mode_name and set_mode_name can all be found in config.js. The mode is stored using Chrome's storage API.

If user wants to change the proxy server address, the extension also provides convenient way to do so. With option-related js, users can change the default proxy server address.

setup_mode_settings does two things - modify the header if needed to bypass the geo filtering, and construct a proxy to direct the traffic to the dedicated proxy server if necessary.

### Header Modification

```Javascript
function setup_mode_settings(mode_name) {
    ...
    case 'normal':
        setup_header();
        setup_proxy();
        chrome.browserAction.setBadgeText({text: ''});
        change_browser_icon('normal');
    case 'off':
    ...
}
```

setup_header essentially assigns a random IP address for any given request when a user is accessing certain URLs:

```
function setup_header() {
    chrome.webRequest.onBeforeSendHeaders.addListener(
        header_modifier,
        {
            // Only applicable when the URL is among unblock_youku.header_urls
            urls: unblock_youku.header_urls
        },
        ['requestHeaders', 'blocking']
    );
}

// Insert the Client-IP and X-Forwarded-For header to fake the IP address.
function header_modifier(details) {
    console.log('modify headers of ' + details.url);

    details.requestHeaders.push({
        name: 'X-Forwarded-For',
        // unblock_youku.ip_addr is a fake IP served by China Telecom.
        value: unblock_youku.ip_addr
    }, {
        // Also modify Client-IP as sometimes this is used to detect the original IP.
        name: 'Client-IP',
        value: unblock_youku.ip_addr
    });

    return {requestHeaders: details.requestHeaders};
}
```

What are X-Forwarded-For and Client-IP here?
X-Forwarded-For was introduced by Squid to identify the originating IP address connecting to a web server through a HTTP proxy. Here its value is a valid random IP address in China. This change hides the original IP address. The Client-IP header is sometimes used to detect the IP address, so set it to the fake address too.

When a request arrives, if the extension detects that the request URL is among the IPs that just changing headers of it is sufficient to bypass the blocking, then it modifies the headers.


## Chrome Proxy Setting Change
Another piece of the Normal mode setting is chaning the Chrome proxy settings.

```Javascript
function setup_proxy() {
    // Overwrite Chrome's proxy settings using a pac script constructed through
    // proxy server info and the URLs we want to send to the proxy.
    function setup_pac_data(proxy_prot_1, proxy_addr_1,
                            proxy_prot_2, proxy_addr_2) {
        var pac_data = urls2pac(
            // The urls defined in url.js. Requests of these URLs will go through the proxy servers in China.
            unblock_youku.chrome_proxy_bypass_urls,
            unblock_youku.chrome_proxy_urls,
            proxy_addr_1, proxy_prot_1,
            proxy_addr_2, proxy_prot_2);
        var proxy_config = {
            mode: 'pac_script',
            pacScript: {
                data: pac_data
            }
        };
        chrome.proxy.settings.set(
            {
                value: proxy_config,
                scope: 'regular'
            },
            function() {}
        );
    } 
    // server info defined in config.js.
    var proxy_server_proc = unblock_youku.default_proxy_server_proc;
    var proxy_server_addr = unblock_youku.default_proxy_server_addr;
    var backup_proxy_server_proc = unblock_youku.backup_proxy_server_proc;
    var backup_proxy_server_addr = unblock_youku.backup_proxy_server_addr;

    // User can set custom proxy. This can be done using the extension's options page (just right click the extension icon). The options are persisted in localStorage.custom_proxy_server_proc and localStorage.custom_proxy_server_addr.
    if (typeof localStorage.custom_proxy_server_proc !== 'undefined' &&
            typeof localStorage.custom_proxy_server_addr !== 'undefined') {
        proxy_server_proc = localStorage.custom_proxy_server_proc;
        proxy_server_addr = localStorage.custom_proxy_server_addr;
        backup_proxy_server_proc = localStorage.custom_proxy_server_proc;
        backup_proxy_server_addr = localStorage.custom_proxy_server_addr;
    }

    setup_pac_data(proxy_server_proc, proxy_server_addr,
                   backup_proxy_server_proc, backup_proxy_server_addr);
}
```

The real meat here is the urls2pac. It constructs a PAC script on the fly. The PAC script will then be used for redirecting certain URLs to the proxy server. With this setup, certain URLs

What is the PAC script like? It basically has two pieces, a regex mapping, and a piece of JS code for redirecting the URL. Here is a small portion of the mapping. It basically says if the URL is like 'v.youku.com/player' or 'v.youku.com/v_show', then we send the traffic to our proxy server. If a URL cannot be matched to any of the patterns, then the request will not go through the proxy server. FindProxyForURL is called. regex matching, and then send it to proxy server by default is
'HTTPS secure.uku.im:8443; HTTPS secure.uku.im:993; DIRECT;'

```
    'v.youku.com': [
      /^\/player\//i,
      /^\/v_show\//i
    ],
    ...
```

## Setting Up Proxy Server
We now know how Unblock-Youku extension works - the extension captures certain URLs that are key to geo location verification, and send these requests to a dedicated proxy server. Now the question is, what is the magic of the proxy server? Fortunately, we can also find the full proxy server configuration in Squid.

Some example URLs that are allowed by the squid proxy. For example,

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

The full list of URLs is [here](https://github.com/uku/Unblock-Youku/wiki/%E5%9C%A8%E7%BE%8E%E5%9B%A2%E4%BA%91%E6%9E%B6%E8%AE%BE%E8%87%AA%E5%B7%B1%E7%9A%84-Unblock-Youku-%E4%BB%A3%E7%90%86%E6%9C%8D%E5%8A%A1%E5%99%A8).

The Squid script sets "forwarded_for transparent", which does not append anything to X-Forwarded-For. Also, it only allows URLs in the crx_url list and rejects all other URLs.

```
# tested for Squid 3.3.8 on CentOS 7.1.1503 64-bit
// The URL list allowed by Squid.
acl crx_url url_regex -i "/opt/crx_url_list.txt"
acl localnet src 10.0.0.0/8 # RFC1918 possible internal network
acl localnet src 172.16.0.0/12  # RFC1918 possible internal network
acl localnet src 192.168.0.0/16 # RFC1918 possible internal network
acl localnet src fc00::/7       # RFC 4193 local private network range
acl localnet src fe80::/10      # RFC 4291 link-local (directly plugged) machines
acl SSL_ports port 443
acl Safe_ports port 80      # http
acl Safe_ports port 443     # https
acl Safe_ports port 1025-65535  # unregistered ports
acl CONNECT method CONNECT
http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports
http_access deny manager
http_access deny to_localhost
http_access allow crx_url
http_access deny all
http_port 8888
cache_dir ufs /var/spool/squid 2048 16 256
coredump_dir /var/spool/squid
refresh_pattern .       0   20% 4320
via off
request_header_access X-Sogou-Auth deny all
request_header_access X-Sogou-Timestamp deny all
request_header_access X-Sogou-Tag deny all
request_header_access Server deny all
request_header_access WWW-Authenticate deny all
request_header_access All allow all
httpd_suppress_version_string on
visible_hostname localhost
forwarded_for transparent
```

## Useful Tips
When exploring the code, I found the following tips very useful:

1. Often times we want to check the logs in the background page. To do so, enable developer mode. On the extensions you can see "inspect views: background" page.
2. chrome://net-internals/#proxy tells you the Chrome proxy setting. The net-internals also tells you many other things.

## Remaining Questions
This blog only investigates a portion of Unblock-Youku features. There are still some interesting questions remaining. For example, how does the redirection mode work, and how we can change DNS to access the Geo-restricted sites? I will address them in some follow-up posts in the future.
