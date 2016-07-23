---
layout: post
title: "Linux平台下的网络工具"
description: ""
category: network
tags: []
---
{% include JB/setup %}

## 常用工具汇总

  + ip

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

  + ifconfig

    此工具包含于 net-tools 中，用于配置网卡设备，已过时。

  + ping

    利用 icmp 回显报文来确定指定主机是否可达的网络检测工具。
	我们经常听人说，某某台机器ping得通，某某台ping不通。
	这里通常暗含这样的意思：ping得通指主机可用，ping不通指主机不可用、挂掉了。
	事实上，ping不通即表示主机不可用的逻辑只在大部分情况下正确。
	少数时候，服务器会通过防火墙过滤掉icmp回显报文，这使得该主机永远不可被ping通，但它实际上又是可用的。

	所以，ping命令只能确定主机是否"在线"，而不能确定主机是否"不在线"。
	当出现ping不通的情况时，我们应利用更为强大的工具来确定主机是否真的"不在线"，例如，nmap。

  + nmap

  + netstat

    用于显示：网络连接、路由表、网卡统计等信息，包含于 net-tools 中。

  + route

  + traceroute

  + tcpdump

  + nslookup/dig/host
