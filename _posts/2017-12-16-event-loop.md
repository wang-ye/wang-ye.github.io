---
layout: post
title:  "Event Loop IO"
date:   2017-12-16 19:59:03 -0800
---

## What is event loop?
Event loop is a power paradigm in programming. In this post we will discuss what it is and how we can implement it in Python. From [Wikipedia](https://en.wikipedia.org/wiki/Event_loop),

>In computer science, the event loop is a programming construct that waits for and dispatches events or messages in a program. It works by making a request to some internal or external "event provider" (that generally blocks the request until an event has arrived), and then it calls the relevant event handler ("dispatches the event").

With the help of low-level select, event loop can detect the files or sockets in ready state and pick the corresponding handler to conduct proper actions. We will investigate a simple echo UDP server in [Python Cookbook](https://github.com/dabeaz/python-cookbook/tree/master/src/11/event_driven_io_explained).

## A Concrete Example
We need to build two components - the *loop* and *event handler*.

```python
class EventHandler:
    def fileno(self):
        'Return the associated file descriptor'
        raise NotImplemented('must implement')

    def wants_to_receive(self):
        'Return True if receiving is allowed'
        return False

    def handle_receive(self):
        'Perform the receive operation'
        pass

    def wants_to_send(self):
        'Return True if sending is requested' 
        return False

    def handle_send(self):
        'Send outgoing data'
        pass

import select

def event_loop(handlers):
    while True:
        wants_recv = [h for h in handlers if h.wants_to_receive()]
        wants_send = [h for h in handlers if h.wants_to_send()]
        can_recv, can_send, _ = select.select(wants_recv, wants_send, [])
        for h in can_recv:
            h.handle_receive()
        for h in can_send:
            h.handle_send()
```

The whole event loop relies on the [*select* call](https://docs.python.org/3/library/select.html). So what is it? From Python 3 documentation:
>This module provides access to the select() and poll() functions available in most operating systems, devpoll() available on Solaris and derivatives, epoll() available on Linux 2.5+ and kqueue() available on most BSD. Note that on Windows, it only works for sockets; on other operating systems, it also works for other file types (in particular, on Unix, it works on pipes). It cannot be used on regular files to determine whether a file has grown since it was last read.

The select module relies on OS specific implementations. For Linux, the man page shows the following:
> select allows a program to monitor multiple file descriptors, waiting until one or more of the file descriptors become "ready" for some class of I/O operation (e.g., input possible).  A file descriptor is considered ready if it is possible to perform a corresponding I/O operation (e.g., read(2) without blocking, or a sufficiently small write(2)).

So in short, the *select* method monitors the state of multiple file descriptors. It is blocking call and returns descriptors in *ready* state. The following example illustrates event-looped based UDP servers.

```python
import socket
import time

from eventhandler import EventHandler, event_loop

class UDPServer(EventHandler):
    def __init__(self, address):
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        self.sock.bind(address)

    def fileno(self):
        return self.sock.fileno()

    def wants_to_receive(self):
        return True

class UDPTimeServer(UDPServer):
    def handle_receive(self):
        msg, addr = self.sock.recvfrom(1)
        self.sock.sendto(time.ctime().encode('ascii'), addr)

class UDPEchoServer(UDPServer):
    def handle_receive(self):
        msg, addr = self.sock.recvfrom(8192)
        self.sock.sendto(msg, addr)

if __name__ == '__main__':
    handlers = [ UDPTimeServer(('',14000)), UDPEchoServer(('',15000))  ]
    event_loop(handlers)
```

## Some Testing
To have a simple test, run:

```shell
echo 'hello' | nc -u localhost 14000
```

You will see the following response:

```shell
Sun Dec 17 20:13:46 2017
```

## Summary
We walked through the basics of event-loop based IO. It utilizes OS level *select* method to monitor multiple file descriptors at the same time, and conduct proper actions when a descriptor is in *ready* state. It is powerful construct widely used in many languages, such as Node.