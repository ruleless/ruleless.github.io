---
layout: post
title: "I/O多路转接(I/O复用)"
description: ""
category: Unix环境编程
tags: [I/O]
---
{% include JB/setup %}

I/O多路转接模型提供一种等待多个描述符就绪的方式，相比于阻塞式I/O模型阻塞于特定描述符的read或write调用，I/O多路转接能监控多个描述符。

以TCP服务器为例：

使用阻塞式I/O时，进程阻塞于accept调用直到内核为监听套接字维护的已完成连接队列不为空，
此时如果只有一个进程（或线程），那么其他已连接客户端则不能得到及时的响应。
所以就必须为每个已连接的客户端创建一个进程（或线程）来保证每个客户端都得到及时的响应。
此方法的不足是：

  1. 进程的创建和调度都需占用系统资源；
  2. 一般而言系统能创建的最大进程数是有限制的（C10K问题）。

使用I/O多路转接时，进程阻塞于select调用直到一个或多个描述符准备就绪。
这就允许我们同时监控监听描述符和已连接描述符，用一个进程为多个客户端提供及时服务也成为可能。

## select系统调用

``` c++
#include <sys/select.h>
#include <sys/time.h>

struct timeval {
	long tv_sec;
	long tv_usec;
};
int select(int maxfdp1, fd_set *readfset, fd_set *writefset, fd_set *exceptionfset, const struct timeval *tv);
// 成功返回就绪的描述符个数，超时返回0，出错返回-1

// fd_set操作函数
void FD_ZERO(fd_set *fdset);
void FD_SET(int fd, fd_set *fdset);
void FD_CLR(int fd, fd_set *fdset);
int  FD_ISSET(int fd, fd_set *fdset);
```

套接字描述符就绪条件：

  + **接收缓冲区中的数据>=接收缓冲区低水位标记** 读就绪
  + **关闭连接的读一半** 读就绪
  + **已完成连接队列非空** 读就绪
  + **发送缓冲区可用空间大小>=发送缓冲区低水位标记** 写就绪
  + **关闭连接的写一半** 写就绪
  + **待处理错误** 读写就绪
  + **TCP带外数据** 异常事件就绪

另外超时参数需要在每次调用select前重置，否则，在第一次超时返回后，select调用将不再等待固定时间而直接返回。

## poll函数

poll提供的功能与select类似。

``` c++
#include <poll.h>

struct pollfd {
	int fd;
	int events;
	int revents;
};
int poll(struct pollfd *fdarray, unsigned long arraysize, int timeout);
// 成功返回就绪的描述符个数，超时返回0，出错返回-1
// fdarray数组中fd为负值的项将被忽略
```

events和revents常值说明：

  + `POLLIN`

    + 是否可为events输入：是
	+ 是否可为revents输出：是
	+ 说明：普通或优先级带数据可读

  + `POLLRDNORM`

    + 是否可为events输入：是
	+ 是否可为revents输出：是
	+ 说明：普通数据可读

  + `POLLRDBAND`

    + 是否可为events输入：是
	+ 是否可为revents输出：是
	+ 说明：优先级带数据可读

  + `POLLPRI`

    + 是否可为events输入：是
	+ 是否可为revents输出：是
	+ 说明：高优先级数据可读

  + `POLLOUT`

    + 是否可为events输入：是
	+ 是否可为revents输出：是
	+ 说明：普通数据可写

  + `POLLWRNORM`

    + 是否可为events输入：是
	+ 是否可为revents输出：是
	+ 说明：普通数据可写

  + `POLLWRBAND`

    + 是否可为events输入：是
	+ 是否可为revents输出：是
	+ 说明：优先级带数据可写

  + `POLLERR`

    + 是否可为events输入：否
	+ 是否可为revents输出：是
	+ 说明：错误

  + `POLLHUP`

    + 是否可为events输入：否
	+ 是否可为revents输出：是
	+ 说明：挂起

  + `POLLNVAL`

    + 是否可为events输入：否
	+ 是否可为revents输出：是
	+ 说明：描述符不是一个打开的文件

poll识别三类数据：普通、优先级带(prioriy band)、高优先级(high priority)，这些术语出自基于流的实现。

那么上表跟TCP和UPD套接字是怎样的一种映射关系呢？

  + 所有正规TCP、UDP数据被认为是普通数据
  + TCP连接上读入的FIN被认为是普通数据
  + TCP连接存在错误既可认为是普通数据，也可认为是错误。无论是哪种情况，随后的读操作将返回-1。
  + 对于监听套接字，已连接队列非空既可认为是普通数据，也可认为是优先级数据。大多数实现将其视为普通数据

poll的timeout参数：

  + -1(INFTIM) 永远等待
  + 0 立即返回
  + \>0 等待固定时间(单位：ms)

## shutdown函数

此函数用于关闭连接的一半。

``` c++
#include <sys/socket.h>

int shutdown(int sockfd, int howto); // 成功返回0，失败返回-1
```

howto参数：

  + `SHUT_RD` 关闭连接的读一半，清空套接字接收缓冲区中的数据
  + `SHUT_WR` 关闭连接的写一半，套接字发送缓冲区中剩下的数据将被发送，且会发送FIN标记
  + `SHUT_RDWR` 与调用两次shutdown等效：第一次调用指定SHUT_RD，第二次调用指定SHUT_WR

## 示例程序

回射程序服务器

``` c++
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <sys/select.h>
#include <sys/time.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define SA struct sockaddr
#define FD_SIZE 2

static void errsys(const char *msg, int nErr);

int main(int argc, char *argv[]) {
	int listenfd = socket(AF_INET, SOCK_STREAM, 0);
	if (listenfd < 0) {
		errsys("create listen fd err!", errno);
	}

	struct sockaddr_in servaddr;
	servaddr.sin_family = AF_INET;
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	servaddr.sin_port = htons(60000);

	if (bind(listenfd, (const SA*)&servaddr, sizeof(servaddr)) < 0) {
		errsys("bind err!", errno);
	}

	if (listen(listenfd, 5) < 0) {
		errsys("listen err!", errno);
	}

	int maxfd = listenfd;
	fd_set allset, readset;
	FD_ZERO(&allset);
	FD_SET(listenfd, &allset);
	FD_ZERO(&readset);

	int connfds[FD_SIZE];
	memset(connfds, -1, sizeof(connfds));

	static const int s_buffSize = 256;
	char s_recvBuff[s_buffSize];

	for (;;) {
		readset = allset;
		int nready = select(maxfd+1, &readset, NULL, NULL, NULL);
		if (nready < 0){
			errsys("select err!", errno);
		}

		if (FD_ISSET(listenfd, &readset)) {     // 有新客户连入服务器
			int fd = accept(listenfd, NULL, NULL);
			printf("new clients! fd:%d\n", fd);
			if (fd >= 0) {
				int i = 0;
				for (; i < FD_SIZE; ++i) {
					if (connfds[i] < 0) {
						connfds[i] = fd;
						FD_SET(fd, &allset);
						if (fd > maxfd) {
							maxfd = fd;
						}
						break;
					}
				}
				if (i == FD_SIZE) {
					printf("too many clients!\n");
					close(fd);
				}
			}

			if (--nready <= 0) {
				continue;
			}
		}

		// 处理连接套接字
		for (int i = 0; i < FD_SIZE; ++i) {
			int fd = connfds[i];
			if (fd >= 0 && FD_ISSET(fd, &readset)) {
				int n = read(fd, s_recvBuff, s_buffSize);
				if (n == 0) {  // 客户端发送FIN
					connfds[i] = -1;
					FD_CLR(fd, &allset);
					if (maxfd == fd) {
						--maxfd;
					}
				}
				else if (n > 0) {  // 回写
					int writen = write(fd, s_recvBuff, n);
					if (writen != n) {
						printf("write sockfd %d err!write len:%d\n", fd, writen);
					}
				}
				else {     // 此情况不应出现
					errsys("read err!", errno);
				}

				if (--nready <= 0)
					break;
			}
		}
	}

	exit(0);
}

static void errsys(const char *msg, int nErr) {
	const char *s_err = strerror(nErr);
	if (NULL == s_err) {
		s_err = "";
	}
	printf("%s %s\n", msg, s_err);
	exit(1);
}
```

回射程序客户端

``` c++
#include <unistd.h>
#include <errno.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <sys/select.h>
#include <sys/time.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define SA struct sockaddr

static void errsys(const char *msg, int nErr);

int main(int argc, char *argv[]) {
	if (argc < 2) {
		errsys("enter your server ip!", 0);
	}

	int sockfd = socket(AF_INET, SOCK_STREAM, 0);
	if (sockfd < 0) {
		errsys("create sockfd err.", errno);
	}

	struct sockaddr_in servaddr;
	memset(&servaddr, 0, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	servaddr.sin_port = htons(60000);

	if (connect(sockfd, (const SA *)&servaddr, sizeof(servaddr)) < 0) {
		errsys("connect err.", errno);
	}

	fd_set allset, rset;
	FD_ZERO(&allset);
	FD_SET(fileno(stdin), &allset);
	FD_SET(sockfd, &allset);

	int maxfd = fileno(stdin);
	if (sockfd > maxfd) {
		maxfd = sockfd;
	}

	static const int s_buffSize = 256;
	char buff[s_buffSize] = {0};
	bool bEof = false;
	for (;;) {
		rset = allset;

		struct timeval limTim;
		limTim.tv_sec = 5;
		limTim.tv_usec = 0;
		int nReady = select(maxfd+1, &rset, NULL, NULL, &limTim);

		if (0 == nReady) {     // timeout
			if (bEof) {
				printf("this situation should't hanppen. the server fucks me?");
				break;
			}
			else {
				printf("tik tok!\n");
				continue;
			}
		}

		if (FD_ISSET(fileno(stdin), &rset)) {  // 从标准输入读入数据
			int n = read(fileno(stdin), buff, s_buffSize);
			if (n > 0) {
				if (write(sockfd, buff, n) != n) {
					printf("send data err.\n");
					exit(2);
				}
			}
			else {
				bEof = true;
				shutdown(sockfd, SHUT_WR);
			}
			--nReady;
		}

		if (nReady > 0 && FD_ISSET(sockfd, &rset)) {  // 收到服务器回写数据
			int n = read(sockfd, buff, s_buffSize);
			if (n > 0) {
				write(fileno(stdout), buff, n);
			}
			else if (n < 0) {
				errsys("server quit.\n", errno);
			}
			else {
				if (bEof) {
					printf("echo finished.\n");
				}
				else {
					printf("server shutdown.\n");
				}
				break;
			}
			--nReady;
		}
	}

	exit(0);
}

static void errsys(const char *msg, int nErr) {
	const char *s_err = "";
	if (nErr != 0) {
		s_err = strerror(nErr);
	}
	printf("%s %s\n", msg, s_err);
	exit(1);
}
```
