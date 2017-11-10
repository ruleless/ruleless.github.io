---
layout: post
title: "socket fd泄漏排查"
description: ""
category: programming
tags: [programming]
---
{% include JB/setup %}

## 问题确认

首先我们要做的当然是确认是否有泄漏问题，那么怎么来确认呢？
假如，我们的目标程序是一个以TCP方式接入连接的服务程序。
我们可以通过对比进程当前活跃的连接数和当前打开的描述符数目来确认是否存在泄漏。

假如，进程ID为$PID，查看该进程活跃的连接的方式是：

``` shell
[root@liuy fd]# ss -anp | grep $PID
LISTEN     0      128                       *:80                       *:*      users:(("nginx",1334,6),("nginx",1337,6))
ESTAB      0      0            192.168.56.101:80            192.168.56.1:48139  users:(("nginx",1337,13))
ESTAB      0      0            192.168.56.101:80            192.168.56.1:47973  users:(("nginx",1337,3))
ESTAB      0      0            192.168.56.101:80            192.168.56.1:48137  users:(("nginx",1337,11))
ESTAB      0      0            192.168.56.101:80            192.168.56.1:48136  users:(("nginx",1337,10))
ESTAB      0      0            192.168.56.101:80            192.168.56.1:48138  users:(("nginx",1337,12))
```

而查看该进程所有打开的文件描述符的方式是：

``` shell
[root@liuy fd]# ls -ls /proc/$PID/fd
total 0
0 lrwx------ 1 liuy liuy 64 Nov 10 22:41 0 -> /dev/null
0 lrwx------ 1 liuy liuy 64 Nov 10 22:41 1 -> /dev/null
0 l-wx------ 1 liuy liuy 64 Nov 10 22:41 2 -> /usr/local/nginx/logs/error.log
0 lrwx------ 1 liuy liuy 64 Nov 10 22:50 3 -> socket:[11191]
0 l-wx------ 1 liuy liuy 64 Nov 10 22:41 4 -> /usr/local/nginx/logs/access.log
0 l-wx------ 1 liuy liuy 64 Nov 10 22:41 5 -> /usr/local/nginx/logs/error.log
0 lrwx------ 1 liuy liuy 64 Nov 10 22:41 6 -> socket:[9262]
0 lrwx------ 1 liuy liuy 64 Nov 10 22:41 7 -> socket:[9268]
0 lrwx------ 1 liuy liuy 64 Nov 10 22:41 8 -> [eventpoll]
0 lrwx------ 1 liuy liuy 64 Nov 10 22:41 9 -> [eventfd]
```

为方便起见，我们可以用下面的脚本来大致的统计上述两者的数量：

``` shell
#!/bin/sh

PID=${1:-`pidof nginx`}
PID=`echo $PID | awk '{ print $1 }'`

tcpnum=`ss -anp | grep $PID | wc -l`
fdnum=`ls -ls /proc/$PID/fd | grep socket | wc -l`

echo "PID=${PID}: tcpnum=${tcpnum}, fdnum=${fdnum}"
```

只要这两者的数量相差不是太大，我们就可以认定基本没有泄露，反之，就可能存在描述符泄露了。
