In this post I want to discuss some useful networking tools for testing connectivity and debugging common network issues.

## Starting Simple HTTP Server
If you have Python 3 installed, starting a web server is simple. Just run

```shell
python3 -m http.server
```
You now have a simple HTTP server serving your local files.


## Starting TCP Echo Server
To start a simple TCP server that echos everything received, you can of course write a couple lines of Python or Node codes and then run them. However, there is an even simpler way using [``netcat``](http://www.tutorialspoint.com/unix_commands/nc.htm). It is a tool dedicated to build and test TCP and UDP connections. To start a tcp server at port 13240, you can run:

```shell
nc -kl localhost 13240
```

Note the `-k` option here to keep the server alive. This server will print out any message it receives. For example, if we try to send some data to port 13240 at localhost using:

```shell
echo 'hello' | nc localhost 13240
```

You will see the message "hello" in stdout under the server. Telnet is another common tool for checking TCP server connectivity. For TCP ports, it works similarly, but it fails at checking a UDP server. So how should we check the UDP server connectivity? The answer is still using `nc`!

## Start UDP Echo Server and Check Connectivity
You can start a UDP server similarly with `nc`:

```shell
nc -ukl localhost 13240
```

and send the UDP packet with

```shell
echo 'hello' | nc -u localhost 13240
```

## List Open Ports
``nc`` command can also list the open ports. For example, if you want to list the open ports between 10 to 50, then run

```shell
nc -z www.example.com 10–50
```

## lsof - List Open Files
`lsof` is another powerful tool to check the port usage. For example, if you want to know all the open files on TCP port 13240:

```shell
22:16:41 ~ $ lsof -i TCP:13240
COMMAND   PID USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
nc      82117   ye    5u  IPv6 0x9d3337a60f3ff4f7      0t0  TCP localhost:13240 (LISTEN)
nc      82117   ye    7u  IPv6 0x9d3337a60f3fe977      0t0  TCP localhost:13240->localhost:63379 (ESTABLISHED)
telnet  82455   ye    5u  IPv6 0x9d3337a60f3fd837      0t0  TCP localhost:63379->localhost:13240 (ESTABLISHED)
```

## Summary
This post discusses some common network diagnosis tools. We can start a simple HTTP server with Python's builtin module. `netcat` is a powerful tool for both starting simple echo TCP/UDP servers as well as testing the sever connectivity. `lsof` can help us understand the port usage.