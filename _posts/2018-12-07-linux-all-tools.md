---
layout: post
title: "Linux工具汇总"
description: ""
category: linux
tags: []
---
{% include JB/setup %}

## 实时状态监控
sysstat工具包：

  1. sar: collects and reports system activity information
  1. iostat: reports CPU utilization and I/O statistics for disks
  1. tapestat: reports statistics for tapes connected to the system
  1. mpstat: reports global and per-processor statistics
  1. pidstat: reports statistics for Linux tasks (processes)
  1. vmstat: summary information of memory, processes, paging etc.
  1. nfsiostat-sysstat: reports I/O statistics for network file systems
  1. cifsiostat: reports I/O statistics for CIFS file systems

另外，如下工具也比较常用：
  1. top
  1. ps
  1. free
  1. pmap
  1. pstack
  1. lsof

## 程序调试分析工具
Linux下的调试工具如下：
  1. gdb: GNU Debugger
  1. gcore: 给运行中的进程产生堆栈

binutils工具包：
  1. ar
  1. as
  1. gprof
  1. ld
  1. nm
  1. objcopy
  1. objdump
  1. ranlib
  1. readelf
  1. size
  1. strings
  1. strip
  1. addr2line

如下工具也比较常用：
  1. strace: 系统调用追踪
  1. valgrind: 内存分析工具

## 性能分析工具
  1. perf

## 网络状态监控及分析工具
  1. netstat
  1. ss
  1. ifconfig
  1. ip

bind-utils工具包：
  1. arpaname
  1. dig
  1. host
  1. nslookup
  1. nsupdate
