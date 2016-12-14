---
layout: post
title: "文件记录锁"
description: ""
category: Unix环境编程
tags: [unix同步机制]
---
{% include JB/setup %}

记录锁的功能是：当一个进程正在修改文件的某个部分时，它能阻止其他进程修改同一文件区。
从记录锁的功能描述可知：记录锁的操作对象是文件区，操作者是进程。
对Unix系统而言，"记录"这个词是一种误用，因为Unix内核根本没有使用文件记录这种概念。
描述记录锁更适合的术语可能是：字节范围锁。

## 各种Unix系统支持的记录锁形式

  * **FreeBSD** 仅支持建议性锁
  * **Linux** 支持建议性锁和强制性锁
  * **Mac OS X** 仅支持建议性锁
  * **Solaris 9** 支持建议性锁和强制性锁

## 文件记录锁规则

文件记录锁由读锁和写锁组成，读锁为共享锁，写锁为独占锁，这与用于线程同步的读写锁语义相同，并且它们的规则也类似：

  1. 当某块文件区域被读锁锁定时，该块区域的任意部分将对写锁封闭、读锁开放。
  2. 当某块文件区域被写锁锁定时，该块区域的任意部分对读锁、写锁均封闭。

在设置或释放文件上的锁时，系统按要求组合或分裂相邻区。

## fcntl

``` c++
#include <fcntl.h>

// 成功的返回值依赖于cmd，失败则返回-1
int fcntl(int filedes, int cmd, ...);
```

fcntl函数有5种功能：

  1. 复制一个现有的描述符(F_DUPFD)

     `dup(fd)` 等价于 `fcntl(fd, F_DUPFD, 0);`

     `dup2(fd, newfd)` 等价于 `close(newfd); fcntl(fd, F_DUPFD, newfd);`

     若成功返回新复制的文件描述符

  2. 获得/设置文件描述符标记(F_GETFD、F_SETFD)

     当前仅定义了一个文件描述符标记：FD_CLOEXEC

  3. 获得/设置文件状态标志(F_GETFL、F_SETFL)

     `F_GETFL` 返回当前的文件状态标志

     `F_SETFL` 设置文件状态标志，可设置的状态标志包括：O_APPEND、O_NONBLOCK、O_SYNC、O_DSYNC ...

  4. 获得/设置异步I/O所有权(F_GETOWN、F_SETOWN)

  5. 获得/设置文件记录锁(F_GETLK、F_SETLK、F_SETLKW)

## 记录锁设置与获取

记录锁需通过flock结构设置与获取：

``` c++
#include <fcntl.h>

struct flock
{
    short l_type; // 可设置为：F_RDLCK、F_WRLCK、F_UNLCK
    off_t l_start; // 相对于l_whence的起始偏移
    short l_whence; // 可设置为：SEEK_SET、SEEK_CUR、SEEK_END
    off_t l_len; // 加锁或解锁区的长度
    pid_t l_pid; // 需获取锁时，返回持有锁的进程ID
};
```

文件记录锁示例程序：

``` c++
int regRecLock(int fd, int cmd, int type, off_t offset, int whence, off_t len)
{
    struct flock lock;
    lock.l_type = type;
    lock.l_start = offset;
    lock.l_whence = whence;
    lock.l_len = len;
    lock.l_pid = 0;
    return fcntl(fd, cmd, &lock);
}

/* 给指定区域加读锁(不阻塞)
 */
#define regReadRecLock(fd, offset, whence, len) \
    regRecLock(fd, F_SETLK, F_RDLCK, offset, whence, len)

/* 给指定区域加读锁(阻塞)
 */
#define regReadWRecLock(fd, offset, whence, len) \
    regRecLock(fd, F_SETLKW, F_RDLCK, offset, whence, len)

/* 给指定区域加写锁(不阻塞)
 */
#define regWriteRecLock(fd, offset, whence, len) \
    regRecLock(fd, F_SETLK, F_WRLCK, offset, whence, len)

/* 给指定区域加写锁(阻塞)
 */
#define regWriteWRecLock(fd, offset, whence, len) \
    regRecLock(fd, F_SETLKW, offset, whence, len)
```
