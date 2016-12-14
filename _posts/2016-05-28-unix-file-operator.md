---
layout: post
title: "Unix用户态下的文件操作接口"
description: ""
category: Unix环境编程
tags: [unix文件]
---
{% include JB/setup %}

## 基本文件I/O函数

``` c++
#include <unistd.h>

int open(int filedes, int filestate, ...); // 失败返回-1， 成功返回文件描述符
int close(int filedes);
int lseek(int filedes, int offset, int fromwhere); // 随机访问文件，失败返回-1，成功返回文件当前偏移量
int read(int filedes, char *buf, int bufsize); // 失败返回-1，成功则返回实际读入的字节数
int write(int filedes, const char *buf, int bufsize); // 失败返回-1，成功则返回实际写入的字节数
```

调用 `open` 函数时，需要指定文件的打开状态标记：

  + `O_RDONLY` 只读
  + `O_WRONLY` 只写
  + `O_RDWR` 读写
  + `O_APPEND` 当以写标记打开文件时，以追加到文件末尾的方式写文件
  + `O_TRUNC` 当以写标记打开文件时，文件将被截短，这也是默认方式
  + `O_CREAT` 若文件不存在则创建它
  + `O_EXCL` 若同时指定了O_CREAT，而文件已经存在，则会出错（EEXIST）
  + `O_NONBLOCK` 以非阻塞方式打开文件（仅对FIFO、块特殊文件和字符特殊文件有效）

调用 `lseek` 函数时，我们需要指定相对偏移位置：

  + `SEEK_SET` 从文件开始位置计算偏移
  + `SEEK_CUR` 从文件当前位置开始计算偏移
  + `SEEK_END` 从文件末尾开始计算偏移

## 内核用于I/O的数据结构

![](/images/unix/file/file-descriptor.png)

fd标志指文件描述符标志，文件描述符标志指示进程退出时是否会关闭文件描述符。

如果有需要，我们可以复制文件描述符，它们有不同的数值，但都指向同一个文件表。
文件描述符复制函数声明如下：

``` c++
#include <unistd.h>

int dup(int filedes); // 成功则返回复制的文件描述符，失败返回-1
int dup2(int filedes, int filedes1); // 该函数将文件描述符复制为指定值，返回值同上
```

复制后的数据结构如下：

![](/images/unix/file/file-descriptor-dup.png)

## fcntl

``` c++
#include <fcntl.h>

int fcntl(int filedes, int cmd, ...); // 成功的返回值依赖于cmd，失败则返回-1
```

fcntl函数提供5种功能

  + **复制一个现有的描述符(F_DUPFD)**

    `dup(fd)` 等价于 `fcntl(fd, F_DUPFD, 0);`
    `dup2(fd, newfd)` 等价于 `close(newfd); fcntl(fd, F_DUPFD, newfd);`
    若成功返回新复制的文件描述符

  + **获得/设置文件描述符标记(F_GETFD/F_SETFD)**

    当前仅定义了一个文件描述符标记：FD_CLOEXEC

  + **获得/设置文件状态标志(F_GETFL/F_SETFL)**

    F_GETFL：返回当前的文件状态标志
    F_SETFL：设置文件状态标志，可设置的状态标志包括：
    `O_APPEND、O_NONBLOCK、O_SYNC、O_DSYNC ...`

  + **获得/设置异步I/O所有权(F_GETOWN/F_SETOWN)**

  + **获得/设置记录锁(F_GETLK、F_SETLK、F_SETLKW)** 用于文件记录锁

## 示例程序（相当于cat命令）

``` c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>

static const int s_buffSize = 4028;
char buff[s_buffSize];

int main(int argc, char* argv[])
{
    int n = 0;
    while ((n=read(STDIN_FILENO, buff, s_buffSize)) > 0)
    {
        if (write(STDOUT_FILENO, buff, n) != n)
        {
            printf("write error! pid=%d\n", getpid());
            exit(1);
        }
    }
}
```
