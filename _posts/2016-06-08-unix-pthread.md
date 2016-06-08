---
layout: post
title: "Unix下的基础线程知识(pthread)"
description: ""
category: Unix环境编程
tags: [线程]
---
{% include JB/setup %}

线程提供一种进程内的并发模式，使用线程的好处是：

  1. 相比于进程使用管道、消息队列或共享内存等复杂的进程间IPC方式来达到共享数据的目的，
     线程可以更容易的实现数据共享，因为多个线程之间自动地可以访问相同的存储地址空间和文件描述符。

  2. 与信号的异步编程模式相比，线程可以使用同步编程模式，而后者比前者简单得多。

  3. 多线程可以把程序中处理用户输入输出的部分与其他逻辑分开。
     例如，大多数应用的日志持久化逻辑就会放置于一个单独的线程。

## 线程ID

线程ID类型定义为pthread_t，但它不一定是整型数值类型。如：

  * Linux 2.4.22使用无符号长整型数表示pthread_t；
  * Solaris 9将pthread_t定义为无符号整数；
  * FreeBSD 5.2.1和Mac OS X 10.3用一个指向pthread结构的指针来表示pthread_t。

线程ID相关的操作接口定义如下：

``` c++
#include <pthread.h>

// 比较两个线程ID是否相等，若相等则返回非0值，不相等则返回0
int pthread_equal(pthread_t tid1, pthread_t tid2);

// 返回调用者的线程ID
pthread_t pthread_self();
```

## 创建线程

线程创建接口定义如下：

``` c++
#include <pthread.h>

int pthread_create(pthread_t *tid, const pthread_attr_t *attr, void* (*pSink)(void*), void *arg);
```

该函数成功返回0，出错则返回错误码。各参数说明如下：

  * 线程ID通过tid返回
  * attr为线程属性
  * pSink为线程回调(或可称之为线程启动例程)
  * arg为传入线程函数的参数

## 终止线程

终止线程有三种方式：

  1. 线程函数正常执行完毕，终止。此方式可返回终止参数。
  2. 调用pthread_exit终止。此方式也可返回终止参数。
  3. 被其他线程通过pthread_cancel终止。此方式返回PTHREAD_CANCELED。

相关函数定义如下：

``` c++
#include <pthread.h>

// 终止本线程，可通过pthread_join查看设置的终止状态。成功返回0，失败返回错误码。
int pthread_exit(void *status);

// 取消ID为tid的其他线程。成功返回0，失败返回错误码。
int pthread_cancel(pthread_t tid);
```

## 等待线程终止

``` c++
#include <pthread.h>

// 成功返回0，出错返回错误码。status返回线程终止参数。
int pthread_join(pthread_t tid, void **status);
```

## 线程终止清理函数

在线程终止之前我们可以执行一个或多个自选函数，大多数时候这是没有必要的。
不过某些场合，我们将看到此种机制的合理以及不可或缺性。
比如，获取了互斥锁的线程被其他线程意外终止。

``` c++
#include <pthread.h>

// 压入清理函数。成功返回0，失败返回错误码。
int pthread_cleanup_pop(int execute);

// 清除清理函数。当execute为0时，只清除函数而不调用它，反之，则否。成功返回0，失败返回错误码。
int pthread_cleanup_push(void* (*pCleanFunc)(void *), void *arg);
```
