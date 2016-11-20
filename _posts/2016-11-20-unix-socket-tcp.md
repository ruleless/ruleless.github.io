---
layout: post
title: "TCP套接字API"
description: ""
category: Unix环境编程
tags: [bsd socket]
---
{% include JB/setup %}

TCP是面向连接的、可靠的字节流协议。
此文档列出的API为创建基于TCP通信的应用程序所必须的API，而并非只能应用于TCP套接字编程。

## 建立TCP连接的三次握手

![](/images/unix/socket/shakehands.png)

## 套接字API

### 套接字描述符创建函数

``` c++
#include <sys/scoket.h>

// 成功返回套接字描述符，失败返回-1
int socket(int family, int type, int protocol);
```

该函数默认创建的为主动套接字，使用listen可将其变为被动套接字。

  + family取值：

    - `AF_INET` 对应IPv4地址族
	- `AF6_INET` 对应IPv6地址族

  + type取值：

    - `SOCK_STREAM` 字节流套接字
	- `SOCK_DGRAM` 数据报套接字
	- `SOCK_SEQPACKET` 有序分组套接字
	- `SOCK_RAW` 原始套接字

  + protocol取值：

    - `IPPROTO_TCP` TCP传输协议
	- `IPPROTO_UDP` UDP传输协议
	- `IPPROTO_SCTP` SCTP传输协议

### 地址绑定函数

``` c++
#include <sys/socket.h>

// 成功返回0，失败返回-1
int bind(int sockfd, const struct sockaddr *addr, uint32_t socklen);
```

该函数绑定一个本地协议地址到套接字。
对于网际网协议，协议地址是32位的IPv4地址或128位IPv6地址与16位的TCP或UDP端口的组合。

### 监听函数

``` c++
#include <sys/socket.h>

int listen(int sockfd, int backlog); // 成功返回0，失败返回-1
```

该函数将一个未连接的主动套接字变为被动套接字。(也可称被动套接字为监听套接字)

内核会为每个监听套接字维护两个队列：

  1. 未完成连接队列，此时已收到客户SYN消息，相应连接处于SYN_RCVD状态；
  2. 已完成连接队列，accept调用将会从此队列中取连接套接字，该队列为空时，accept阻塞，反之，则返回。

backlog参数表示已完成连接队列的最大长度。
早先版本中，该参数表示未完成连接队列加已完成队列的最大长度。
考虑这样一种情况：
未完成连接队列被恶意客户塞满(即只向服务器发送SYN同步分节而不确认服务器的SYN分节)，
那么已完成队列将永远为空。这是一种早期的拒绝服务攻击形式。

### 请求连接函数

``` c++
#include <sys/socket.h>

// 成功返回0，失败返回-1
int connect(int sockfd, const struct sockaddr *serveraddr, uint32_t socklen);
```

在阻塞模式下该函数将导致应用进程阻塞直到服务器响应。
此函数将会触发三次握手的发生。
它首先向服务器发送一个同步分节(SYN)，然后等待，
若无回应则在6s后再次发送SYN，若再无回应则在24s后继续发送。
在服务器回应确认分节(ACK)并顺带发送了一个同步分节(SYN)之后，该函数回应服务器一个ACK并返回。

### accept函数

``` c++
#include <sys/socket>

// 成功返回连接描述符，出错返回-1
int accept(int sockfd, struct sockaddr *cliaddr, uint32_t *addrlen);
```

该函数在已完成连接队列为空时阻塞，非空时返回。

### 获取与套接字关联的协议地址

``` c++
#include <sys/socket.h>

// 返回跟套接字关联的本地协议地址
int getsockname(int sockfd, struct sockaddr *localaddr, uint32_t *addrlen);

// 返回跟套接字关联的外地协议地址
int getpeername(int sockfd, struct sockaddr *peeraddr, uint32_t *addrlen);
```

什么情况下需要上述函数？

  1. 在连接套接字上调用accept只能返回客户的地址，调用getsockname才能获取本机与客户建立连接的本地地址
  2. 当服务器调用fork创建子进程处理客户业务时，子进程获得客户身份的唯一途径是getpeername

## 示例程序

回射程序服务器端：

``` c++
#include <unistd.h>
#include <signal.h>
#include <sys/wait.h>
#include <errno.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define SA struct sockaddr

static void errsys(const char *errmsg);
static void sigChild(int signo);
static void echoClient(int connfd, const SA *cliaddr, uint32_t addrlen);

int main(int argc, char *argv[])
{
	int listenfd = -1;
	if ((listenfd = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
		errsys("create listen sock err!");
	}

	struct sockaddr_in servaddr;
	servaddr.sin_family = AF_INET;
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);

	servaddr.sin_port = htons(60000);
	if (bind(listenfd, (SA*)&servaddr, sizeof(servaddr)) < 0) {
		errsys("bind addr err!");
	}

	if (listen(listenfd, 5) < 0) {
		errsys("listen failed!");
	}

	struct sigaction newact, oldact;
	newact.sa_handler = sigChild;
	sigemptyset(&newact.sa_mask);
	newact.sa_flags = SA_INTERRUPT;
	if (sigaction(SIGCHLD, &newact, &oldact) < 0) {
		errsys("setup signal failed!");
	}

	for (;;) {
		int connfd = -1;
		struct sockaddr_in cliaddr;
		uint32_t addrLen = sizeof(cliaddr);

		if ((connfd = accept(listenfd, (SA *)&cliaddr, &addrLen)) < 0) {
			if (errno == EINTR) {
				continue;
			}
			else {
				errsys("accept err!");
			}
		}
		else {
			int pid = -1;
			if ((pid = fork()) < 0) {
				errsys("fork err!");
			}
			else if (pid == 0) {
				close(listenfd);
				echoClient(connfd, (const SA *)&cliaddr, addrLen);
				exit(0);
			}

			close(connfd);
		}
	}
	exit(0);
}

static void errsys(const char *errmsg) {
	printf("%s\n", errmsg);
	exit(1);
}

static void sigChild(int signo) {
	int status = 0;
	int pid = wait(&status);

	static const int s_buffSize = 255;
	char hintBuff[s_buffSize];
	snprintf(hintBuff, s_buffSize-1, "child process %d exit status:%d\n", pid, status);
	write(STDIN_FILENO, hintBuff, strlen(hintBuff));
}

static void echoClient(int connfd, const SA *cliaddr, uint32_t addrlen) {
	static const int s_buffSize = 1024;
	char buff[s_buffSize] = {0};

	for (;;) {
		int n = 0;
		if ((n = read(connfd, buff, s_buffSize)) < 0) {
			if (errno == EINTR) {
				continue;
			}
			else {
				errsys("server read err!");
			}
		}
		else if (n > 0) {
			if (write(connfd, buff, n) != n) {
				errsys("write err!");
			}
		}
		else {
			close(connfd);
			break;
		}
	}
}
```

回射程序客户端：

``` c++
#include <unistd.h>
#include <sys/socket.h>
#include <errno.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define SA struct sockaddr

static void errsys(const char *msg);

int main(int argc, char *argv[]) {
	static const int s_errBuffSize = 256;
	char errBuff[s_errBuffSize] = {0};
	if (argc < 2) {
		errsys("no input param!");
	}

	int cliFD = -1;
	if ((cliFD = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
		errsys("create socket err!");
	}

	struct sockaddr_in servaddr;
	servaddr.sin_family = AF_INET;
	if (inet_pton(AF_INET, argv[1], &servaddr.sin_addr) <= 0) {
		errsys("illegal ip addr!");
	}
	servaddr.sin_port = htons(60000);

	if (connect(cliFD, (const SA *)&servaddr, sizeof(servaddr)) < 0) {
		snprintf(errBuff, s_errBuffSize, "connect err! %s", strerror(errno));
		errsys(errBuff);
	}

	static const int s_buffSize = 1024;
	char buff[s_buffSize];
	while (fgets(buff, s_buffSize, stdin) != NULL) {
		int n = strlen(buff);
		if (write(cliFD, buff, n) != n) {
			errsys("write err!");
		}

		if ((n = read(cliFD, buff, s_buffSize)) < 0) {
			errsys("read err!");
		}
		else if (n > 0) {
			write(STDOUT_FILENO, buff, n);
		}
		else {
			snprintf(buff, s_buffSize, "server shutdown!");
			write(STDOUT_FILENO, buff, strlen(buff));
			break;
		}
	}

	exit(0);
}

static void errsys(const char *errmsg) {
	printf("%s\n", errmsg);
	exit(1);
}
```

说明：

上述示例是采用阻塞I/O模式的简单回射程序。
服务器用的是并发模式，每个客户端都有一个独立的进程为其服务。
但实际应用中并不常采用这种方式，因为进程属于操作系统稀有资源，
而且在客户端较多的时候，进程调度也是一笔不小的开销。
实际应用中常用到的是 **I/O复用+多线程模式** 或 **I/O复用+非阻塞式I/O模式** 。
