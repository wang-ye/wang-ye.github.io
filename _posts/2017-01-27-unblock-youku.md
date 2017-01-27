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
proxy server in China - set up through https://github.com/uku/Unblock-Youku/wiki/%E5%9C%A8%E7%BE%8E%E5%9B%A2%E4%BA%91%E6%9E%B6%E8%AE%BE%E8%87%AA%E5%B7%B1%E7%9A%84-Unblock-Youku-%E4%BB%A3%E7%90%86%E6%9C%8D%E5%8A%A1%E5%99%A8

squid

light mode:
no proxy needed.
Modify http headers + redirect the IP address checking request to Unblock-Youku's own proxy.
.