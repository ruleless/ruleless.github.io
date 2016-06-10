---
layout: post
title: "管道、FIFO"
description: ""
category: Unix环境编程
tags: [进程IPC]
---
{% include JB/setup %}

管道是最初的Unix IPC形式，尽管对于许多操作系统来说管道很有用，但它的根本局限性在于没有名字，从而只能由有亲缘关系的进程使用。
这一点随FIFO的加入在System III Unix中得以改正。FIFO有时成为有名管道。
管道和FIFO都使用常用的read和write来进行访问。

## 管道

``` c++
#include <unistd.h>

int pipe(int fds[2]); // 成功返回0，失败返回-1
```

该函数返回两个文件描述符：fd[0]和fd[1]，前者用于读，后者用于写。

单进程利用管道作数据交互的示意图：

![](/images/unix/ipc/pipe-1.png)

多进程利用管道作单向数据交互的示意图：

![](/images/unix/ipc/pipe-2.png)

上图给出了一个在父子进程中利用管道进行单向数据通信的示例。
首先，由父进程创建一个管道后调用fork派生一个自身的副本；
接着，父进程关闭这个管道的读出端，子进程关闭这个管道的写入端。

多进程利用管道作双向数据交互的示意图：

![](/images/unix/ipc/pipe-3.png)

父子进程利用管道进行双向数据传输需要创建两个管道，如上图所示。
然后，我们在父进程中关闭管道1的写入端、管道2的读出端；
在子进程中关闭管道1的读出端、管道2的写入端。

我们在某个Unix shell中输入一个像下面这样的命令时：

``` shell
who | sort | lp
```

该shell将创建三个进程和其间的两个管道。
它还把每个管道的读出端复制到相应进程标准输入，把每个管道的写入端复制到相应进程的标准输出。
其管道线如下：

![](/images/unix/ipc/pipe-4.png)

### popen和pclose

``` c++
#include <stdio.h>

// 成功则返回文件指针，失败则返回NULL
FILE* popen(const char *command, const char *type);

// 成功则返回shell的终止状态，失败则返回-1
int pclose(FILE *stream);
```

  + command参数是一个shell命令行，它是由shell程序处理的；
  + type可为r或w，当为r时：管道的写入端被复制为command进程的标准输出；
    当为w时，管道的读出端被复制为command进程的标准输入。

## FIFO(named pipe)

``` c++
#include <sys/types.h>
#include <sys/stat.h>

int mkfifo(const char *pathname, mode_t mode); // 成功返回0，失败返回-1
```

在创建一个FIFO后，该FIFO必须或者打开来读，或者打开来写，FIFO不能以读写方式打开，因为它是半双工的。
对管道或FIFO的write总是往末尾添加数据，对它们的read则总是从开头返回数据。
如果对管道或FIFO调用lseek，将会返回ESPIPE错误。

## O_NONBLOCK对管道和FIFO的影响

  + 当前操作：open FIFO(只读)

    - 现有打开操作：FIFO打开来写

      * 阻塞模式：成功返回
	  * 非阻塞模式：成功返回

	- 现有打开操作：FIFO不是打开来写

	  * 阻塞模式：阻塞到FIFO打开来写为止
	  * 非阻塞模式：成功返回

  + 当前操作：open FIFO(只写)

    - FIFO打开来读

	  * 阻塞模式：成功返回
	  * 非阻塞模式：成功返回

	- FIFO不是打开来读

	  * 阻塞模式：阻塞到FIFO打开来读为止
	  * 非阻塞模式：返回ENXIO

  + 当前操作：从空管道或空FIFO read

    - 管道或FIFO打开来写

	  * 阻塞模式：阻塞到管道或FIFO中有数据或者管道或FIFO不再为写打开着为止
	  * 非阻塞模式：返回EAGAIN

	- 管道或FIFO不是打开来写

	  * 阻塞模式：read返回0（文件结束符）
	  * 非阻塞模式：read返回0（文件结束符）

  + 当前操作：往管道或FIFO write

	- 管道或FIFO打开来读

	  * 阻塞模式：（见下面描述）
	  * 非阻塞模式：（见下面描述）

	- 管道或FIFO不是打开来读

	  * 阻塞模式：给线程产生SIGPIPE信号
	  * 非阻塞模式：给线程产生SIGPIPE信号

关于管道或FIFO的读出与写入操作的若干规则描述如下：

  + 如果请求读出的数据量多于管道或FIFO中当前可用的数据量，那么只返回这些可用的数据；

  + 如果请求写入的数据量小于或等于PIPE_BUF，那么write操作将保证是原子的。
    这意味着，如果有两个不同的进程差不多同时往一个管道或FIFO写数据，
	那么，或者先写入来自第一个进程的所有数据，然后再写入来自第二个进程的所有数据；
	或者颠倒过来。系统不会相互混杂来自这两个进程的数据。
	但是，如果请求写入的数据大于PIPE_BUF，那么write操作将不能保证是原子的。

    O_NONBLOCK标志的设置对write操作的原子性没有影响（原子性由且仅由所请求写入的字节数是否<=PIPE_BUF所决定）。
	然而，当一个管道或FIFO设置成非阻塞时，write的返回值取决于待写入的数据以及该管道或FIFO当前的可用空间大小。

    - 如果待写的数据<=PIPE_BUF：

	  * 如果待写入字节数<=当前管道或FIFO可用空间大小，那么所有数据写入；
	  * 如果待写入字节数>当前管道或FIFO可用空间大小，那么立即返回EAGAIN错误。

    - 如果待写的数据>PIPE_BUF：

	  * 如果该管道或FIFO至少有1字节空间，那么内核写入该管道或FIFO能容纳的最大数目的字节，该数目同时作为write的返回值；
	  * 如果该管道或FIFO已满，则返回EAGAIN错误。

## 示例程序(并发模式下的回射程序)

common.h

``` c++
#ifndef __COMMON_H__
#define __COMMON_H__

#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define FIFO_PATH "/tmp/ServerFIFO"
#define FIFO_ECHO "/tmp/ECHOFIFO."

#define MAX_BUF 256

extern void errQuit();

#endif
```

common.cpp

``` c++
#include "common.h"

void errQuit()
{
	printf("%s\n", strerror(errno));
	exit(1);
}
```

client.cpp

``` c++
#include "common.h"

int main(int argc, char *argv[])
{
	char echoRPath[MAX_BUF];
	char echoWPath[MAX_BUF];
	snprintf(echoRPath, MAX_BUF, "%s%dr", FIFO_ECHO, (int)getpid());
	snprintf(echoWPath, MAX_BUF, "%s%dw", FIFO_ECHO, (int)getpid());

	if (mkfifo(echoRPath, 0644) < 0 && errno != EEXIST)
		errQuit();
	if (mkfifo(echoWPath, 0644) < 0 && errno != EEXIST)
		errQuit();

	int servFd = open(FIFO_PATH, O_WRONLY);
	if (servFd < 0)
		errQuit();

	char buf[MAX_BUF];
	snprintf(buf, MAX_BUF, "%10d", (int)getpid());
	write(servFd, buf, 10);

	int readFd = open(echoRPath, O_RDONLY);
	if (readFd < 0)
		errQuit();

	int n = read(readFd, buf, 2);
	if (n != 2 || buf[0] != 'o' || buf[1] != 'k')
		errQuit();

	int writeFd = open(echoWPath, O_WRONLY);
	if (writeFd < 0)
		errQuit();

	while (fgets(buf, MAX_BUF, stdin) != NULL)
	{
		int len = strlen(buf);
		write(writeFd, buf, len);

		n = read(readFd, buf, MAX_BUF-1);
		if (n > 0)
		{
			buf[n] = '\0';
			printf("server ret:%s", buf);
		}
	}

	exit(0);
}
```

server.cpp

``` c++
#include "common.h"

int main(int argc, char *argv[])
{
	if (mkfifo(FIFO_PATH, 0644) < 0 && errno != EEXIST)
		errQuit();

	int servFd = open(FIFO_PATH, O_RDONLY);
	if (servFd < 0)
		errQuit();

	int dummyfd = open(FIFO_PATH, O_WRONLY);
	if (dummyfd < 0)
		errQuit();

	char buf[11] = {0};
	int n = 0;
	memset(buf, 0, sizeof(buf));
	while ((n = read(servFd, buf, 10)) > 0)
	{
		if (n != 10)
		{
			printf("client write len illegal len = %d\n", n);
			continue;
		}

		int clientpid = atoi(buf);
		if (clientpid > 0)
		{
			if (fork() == 0)
			{
				char echoRPath[MAX_BUF];
				char echoWPath[MAX_BUF];
				snprintf(echoRPath, MAX_BUF, "%s%dw", FIFO_ECHO, clientpid);
				snprintf(echoWPath, MAX_BUF, "%s%dr", FIFO_ECHO, clientpid);

				int writeFd = open(echoWPath, O_WRONLY);
				if (writeFd < 0)
					errQuit();
				write(writeFd, "ok", 2);

				int readFd = open(echoRPath, O_RDONLY);
				if (readFd < 0)
					errQuit();

				char echoBuf[MAX_BUF];
				int n = 0;
				while((n = read(readFd, echoBuf, MAX_BUF)) > 0)
				{
					write(writeFd, echoBuf, n);
				}

				printf("client %u stoped.\n");

				exit(0);
			}
		}
	}

	exit(0);
}
```
