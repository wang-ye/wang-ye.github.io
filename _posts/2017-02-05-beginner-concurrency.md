---
layout: post
title:  "Random Concurrency Thought"
date:   2017-02-06 21:31:01 -0800
---

In his blog "[Concurrency: A Primer](https://sookocheff.com/post/concurrency/concurrency-a-primer/)", Kevin Sookocheff shared his thoughs towards building concurrent programs. This reminded me of some basic principles when writing concurrent code.

Kevin mentioned that there are three practices to build up concurrency.

1. Using synchronization when accessing mutable shared state: A lot languages provide synchronized or similar keywords to help you write code with explicit locking mechanisms. This is the most flexible, but also the most error-prone approach.
2. Isolating state between threads: this approach handles concurrency at a higher level. Instead of writing explicit locks or synchronized keywords, utilities such as Go’s goroutine functionality and Java’s java.util.concurrent package are used. This makes the coding easier.
3. Making state immutable: With the popularity of functional programming, such as Scala, the use of immutable data structures have become more and more popular. This makes the parallelism especially simple.

The blog also discusses some interesting topics including Futures, Communicating Sequential Processes (Go lang), actor model (what's app) and Software Transactional Memory (Clojure and Haskell). Definitely a fun read!

My biggest take away is, when possible, do not use explicit locks or synchronized keywords!