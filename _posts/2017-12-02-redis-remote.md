---
layout: post
title:  "Configure Redis to Accept Remote Requests"
date:   2017-12-02 20:49:03 -0800
---

I was trying to configure a Redis instance that can be accessed by several Ubuntu servers. We basically need to do two things:

1. Update Redis server config to accept external requests
2. Update iptable rules to accept external requests

Also, all configurations will only happen in the machine with Redis instance.

## Redis Configs
First, we need to install Redis on all Ubuntu servers. I am using Ubuntu 16.04.

```shell
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install redis-server
```

Then, we need to configure Redis to accept external requests. The Redis config file is in */etc/redis/redis.conf*. We want to change the statement **bind 127.0.0.1** to **bind 0.0.0.0**. 

```conf
...
# Comment out 127.0.0.1 and add 'bind 0.0.0.0'.
# bind 127.0.0.1
bind 0.0.0.0
...
```

After this change, we need to restart Redis:

```shell
sudo systemctl restart redis
```

## IPtable Config
On iptable-side, we need to configure [*iptables*](https://gist.github.com/djaiss/ec8cab133fdf03a03f86) to accept external requests:

```shell
# Replace the ip addr to your actual addr.
sudo iptables -A INPUT -p tcp --dport 6379 -s ip_addr -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6379 -j DROP
```

After the iptables change, make sure the rules are indeed added.

```shell
root@ye:~# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
ACCEPT     tcp  --  138.197.91.10        anywhere             tcp dpt:6379

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
```

It is also better to check the port status with *netsat* command. We want to see that the local address to be "0 0.0.0.0:6379" and foreign address to be "0.0.0.0:\*" for the rule listening on *redis-server*.

```shell
root@ye:~# netstat -nlpt
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:6379            0.0.0.0:*               LISTEN      12160/redis-server 
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1325/sshd       
```


Finally, persist them so that the rebooted system can also read the iptable config. You will need to save the iptable config to a file with

```shell
# Save current firewall config
sudo iptables-save > /etc/iptables.conf
```

,and paste the following line to /etc/rc.local.

```
# Inside /etc/rc.local
iptables-restore < /etc/iptables.conf
```

That's it!
