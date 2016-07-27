---
layout: post
title: "Linux平台下的网络工具"
description: ""
category: network
tags: []
---
{% include JB/setup %}

## 常用工具汇总

  + **ifconfig**

    此工具包含于 net-tools 中，用于配置网卡设备，已过时。

  + **ip**

    显示或管理路由表及网卡的工具。我通常用该工具来管理网卡设备，路由表的管理则用 route。
	当以最小工具集的方式安装好操作系统后，网络肯定是未连接的。
	这个时候，你得确定你有哪些网卡，还有各网卡的Mac地址。
	我们可以用 `ip link show` 来显示我们所有的网卡及其相关的设备信息。
	如何启动或关闭某个网卡呢？我们可以用 `ip link set dev(eth0) up/down` 来启动或关闭某个网卡设备。
	但通常情况下，我们不会用这种方式，换而用 `ifup` 和 `ifdown` 命令。
	`ifup` 命令的好处是，它不仅可以启动设备还可以进一步获取设备的IP地址。
	如，`ifup eth0` 相当于是先后执行了命令：`ip link set eth0 up`, `service network start`。

	除了 `ip link`，`ip addr` 也是常用的命令格式。
	`ip link` 用于管理网卡设备；`ip addr` 则用于IP地址管理。
	在使用 `ip` 命令的时候会发现，它有很多跟 `ifconfig` 重复的功能，而且还有很多 `ifconfig` 没有功能。
	事实上，`ifconfig` 是一个过时的工具，`ip` 可以作为它的替代工具来使用。

  + **ping**

    利用 icmp 回显报文来确定指定主机是否可达的网络检测工具。
	我们经常听人说，某某台机器ping得通，某某台ping不通。
	这里通常暗含这样的意思：ping得通指主机可用，ping不通指主机不可用、挂掉了。
	事实上，ping不通即表示主机不可用的逻辑只在大部分情况下正确。
	少数时候，服务器会通过防火墙过滤掉icmp回显报文，这使得该主机永远不可被ping通，但它实际上又是可用的。

	所以，ping命令只能确定主机是否"在线"，而不能确定主机是否"不在线"。
	当出现ping不通的情况时，我们应利用更为强大的工具来确定主机是否真的"不在线"，例如，nmap。

  + **nmap**

  + **netstat**

    用于显示Linux网络子系统相关的信息，包括：网络连接、路由表、网络统计等。包含于 net-tools 中。
	已过时。

  + **ss**

	用于显示和统计sockets信息，是netstat的替代工具。

	不带参数时，ss默认显示已建立连接的TCP管道，ss通常情况下的用到的参数有：

	  + `-a` 显示监听和非监听sockets
	  + `-l` 仅显示监听sockets
	  + `-n` 禁用名字解析
	  + `-p` 显示关联进程
	  + `-t` 显示TCP sockets
	  + `-u` 显示UDP sockets
	  + `-d` 显示DCCP sockets
	  + `-w` 显示RAW sockets
	  + `-x` 显示Unix domain sockets
	  + `-s` 显示统计信息

    常见用法：

	  + `ss -ant` 显示所有TCP sockets
	  + `ss -lnt` 仅显示处于LISTEN状态的TCP sockets
	  + `ss -anu` 显示所有UDP sockets
	  + `ss -anx` 显示所有Unix domain sockets

  + **route**

    显示和管理路由表。

	常见用法：

	  + `route -n` 显示路由表(不进行名字解析)
	  + `route add -net 0.0.0.0 gw 10.0.2.2 dev eth0` 添加一条默认路由
	  + `route add -net 10.0.2.0 netmask 255.255.255.0 dev eth1` 新增一条路由规则：发往10.0.2.x的网络包都走eth1接口
	  + `route del -net 10.0.2.0 netmask 255.255.255.0` 删除一条路由规则

    需要注意的是，gw参数指定的下一跳地址必须与主机在同一网段内。

  + **traceroute**

    追踪并输出网络包到目的主机所经的各路由节点。

  + **tcpdump**

    网络包抓取分析工具。这是分析网络故障时用到频率最高的工具，也是最有效的工具。
	它的常见用法如：

	  + `tcpdump -D` 输出所有可监听接口的序号
	  + `tcpdump icmp` 输出所有的icmp报文
	  + `tcpdump port 1080` 输出所有源端口或目的端口为1080的UDP和TCP数据报
	  + `tcpdump tcp src host 10.0.2.12` 输出所有源主机为10.0.2.12的TCP报文

  + **nslookup/dig/host**

    DNS正、反解析工具，包含于bind-utils中。

## 网络故障排查

如果能熟练使用以上工具，日常所遇到的90%的网络故障我们都可以自己排查，就算不能解决，也起码能知道问题出在哪个环节。
