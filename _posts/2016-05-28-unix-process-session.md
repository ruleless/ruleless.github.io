---
layout: post
title: "进程会话"
description: ""
category: Unix环境编程
tags: [unix进程]
---
{% include JB/setup %}

Unix操作系统调度的基本单位是进程，单个或多个进程组成进程组，单个或多个进程组则组成会话。
其关系示意图如下：

![](/images/unix/process/process-session.png)

## 进程组

跟进程组相关的访问和控制接口声明如下：

``` c++
#include <unistd.h>

pid_t getpgrp(); // 返回调用进程的进程组ID
pid_t getpgid(pid_t pid); // 返回指定进程的进程组ID，若失败返回-1
int setpgid(pid_t pid, pid_t pgid); // 设置指定进程的进程组ID，成功返回0，失败返回-1
```

一个进程组中有一个组长进程和0个或多个组员进程，组长进程的进程ID等于其进程组ID。
所以，可以通过调用 `setpgid(getpid() , getpid())` 创建一个进程组。

## 会话

创建会话的接口函数：

``` c++
#include <unistd.h>

pid_t setsid(); // 若成功则返回进程组ID，出错返回-1
```

如果调用此函数的进程不是组长进程，那么此函数将创建一个新会话。调用此函数将发生：

  1. 该进程成为新会话的首进程，也是新会话中的唯一进程
  2. 该进程成为新进程的组长进程
  3. 若该进程有控制终端，则会与控制终端断开联系

## 控制终端

会话和进程组与控制终端的联系：

  1. 一个会话可以有一个控制终端
  2. 建立与控制终端连接的会话首进程被称为控制进程
  3. 如果一个会话有一个控制终端，则它有一个前台进程组，会话中的其他进程组则称为后台进程组
  4. 中断(SIGINT)和退出(SIGQUIT)信号只会发给会话中的前台进程组

获取和设置会话的控制终端的函数为：

``` c++
#include <unistd.h>

pid_t tcgetpgrp(int filedes); // 成功则返回前台进程组的进程组ID，出错返回-1
int tcsetpgrp(int filedes, pid_t pgid); // 设置前台进程组，成功返回0，失败返回-1
```

## 示例

下面展示系统启动时创建进程组和会话的过程。

系统启动示意图：

![](/images/unix/process/startup.png)
