---
layout: post
title: "Posix 信号量"
description: ""
category: Unix环境编程
tags: [unix同步机制]
---
{% include JB/setup %}

## 信号量概要

信号量是一种用于提供不同进程或线程间同步手段的原语。

1965年荷兰的计算机科学家E.W.Dijkstra提出了用于进程同步的新工具：信号量和PV操作。
他将交通管制中利用多种颜色的信号灯管理交通的方法引入操作系统，让两个或多个进程通过特殊变量进行交互。
一个进程在某一特殊点上被迫停止执行直到接收到一个对应的特殊变量值，
通过特殊变量这一设施，任何复杂的进程交互要求可得到满足，这种特殊变量就是信号量（semaphore）。

为了通过信号量传送信号，进程可以通过P、V 两个特殊的操作来发送和接收信号。
如果进程相应的信号仍然没有送到，进程被挂起直到信号到达为止。
在操作系统中，信号量用以表示物理资源的实体，它是一个与队列有关的整型变量。
实现时，信号量是一种变量类型，常常用一个记录型数据结构表示，
它有两个分量：一个是信号量的值，另一个是信号量队列的队列指针。
除赋初值外，信号量仅能由同步原语对其进行操作，没有任何其他方法可以检查和操作信号量。
原语是操作系统中执行时不可中断的过程、即原子操作。
Dijkstra 发明了两个信号量操作原语： P 操作和 V 操作，
此外，还常用的符号有： wait 和signal, up 和 down, sleep 和 wakeup 等。

利用信号量和 P、 V 操作既可以解决并发进程的竞争问题，又可以解决并发进程的协作问题。

信号量按其用途可分为两种：

  1. 公用信号量： 联系一组并发进程，相关的进程均可在此信号量上执行 PV操作。初值常常为 1 ，用于实现进程互斥。
  2. 私有信号量： 联系一组并发进程，仅允许此信号量拥有的进程执行 P 操作，而其他相关进程可在其上施行 V 操作。初值常常为 0 或正整数，多用于并发进程同步。

信号量按其取值可分为两种：

  1. 二值信号量： 仅允许取值为 0 和 1 ，主要用于解决进程互斥问题。
  2. 计数信号量： 允许取值为非负整数，主要用于解决进程同步问题。

信号量按其实现可分为两种：

  1. 整型信号量
  2. 记录型信号量

**整型信号量简述**

设 s 为一个正整形量，除初始化外，仅能通过 P V 操作来访问它，这时 PV 操作原语定义如下：

  * P(s)：当信号量 s 大于 0 时，把信号量 s 减去1，否则调用 P(s)的进程等待直到信号量 s 大于 0 。
  * V(s)：把信号量 s 加 1 。


P(s)用代码可表述为：

``` c++
whiel (s<=0);
--s;
```

V(s)可表述为：

``` c++
++s;
```

整型信号量机制中的 P 操作，只要信号量 s<=0，就会不断测试，进程处于"忙式等待”。
后来对整型信号量进行了扩充，增加了一个等待 s 信号量所代表资源的等待进程的队列，
以实现让权等待，这就是记录型信号量机制。

**记录型信号量简述**

设 s 为一个记录型数据结构，其中一个分量为整形量value，另一个分量为信号量队列 queue，
value 通常是一个具有非负初值的整型变量， queue 是一个初始状态为空的进程队列。
这时 PV 操作原语的定义如下：

  * P(s)：将信号量 s 减去 1，若结果 < 0，则调用 P(s) 的进程被置成等待信号量 s 的状态。
  * V(s)：将信号量 s 加 1，若结果 <= 0，则释放一个等待信号量 s 的进程。

P(s)用代码可表述为：

``` c++
s.value = s.value – 1;
if (s.value < 0)
{
	W(s.queue);
}
```

V(s)可表述为：

``` c++
s.value = s.value + 1;
if (s.value <= 0)
{
	R(s.queue);
}
```

  * W(s.queue) 表示把调用过程的进程置成等待信号量 s 的状态，并链入 s 信号量队列，同时释放 CPU；
  * R(s.queue) 表示释放一个等待信号量 s 的进程，从信号量 s 队列中移出一个进程，置成就绪态并投入就绪队列。

从信号量和 PV 操作的定义可以获得如下推论：

  1. 若信号量 s > 0，则该值等于在封锁进程之前对信号量 s 可施行的 P 操作数，
     亦即等于 s 所代表的实际还可以使用的物理资源数。
  2. 若信号量 s < 0，则其绝对值等于登记排列在该信号量 s 队列之中等待的进程个数，
     亦即恰好等于对信号量 s 实施 P 操作而被封锁起来并进入信号量 s 等待队列的进程数。
  3. 通常，P 操作意味着请求一个资源，V 操作意味着释放一个资源。
     在一定条件下， P 操作代表挂起进程操作，而 V 操作代表唤醒被挂起进程的操作。

## Posix信号量

Posix有两种类型的信号量：有名信号量、基于内存的信号量。

有名信号量基于文件系统实现，不需要做特殊处理即可用于进程和线程之间的交互；
基于内存的信号量如果需要在进程之间共享，则需要将它分配到共享内存区。

使用Posix的信号量有三个步骤：

  1. 创建一个信号量。sem_open()或sem_init()
  2. 等待一个信号量。sem_wait或sem_trywait()
  3. 挂出一个信号量。sem_post


有名信号量和基于内存的信号量的系统调用函数汇总如下：

![](/images/unix/sync/sem_func.png)

信号量函数：

``` c++
#include <semaphore.h>

// 失败返回SEM_FAILED，函数参数与open()类似
sem_t* sem_open(const char *name, int oflag, .../* mode_t mode, unsigned int value */);

// 该函数不会将信号量从系统中删除
int sem_close(sem_t *s);

// 类似于文件I/O的unlink调用
int sem_unlink(sem_t *s);

// 分配并初始化一个内存信号量
int sem_init(sem_t *s, int shared, unsigned int value);

// 销毁并释放信号量
int sem_destroy(sem_t *s);

// 相当于P操作原语
int sem_wait(sem_t *s);

// 当所指定的信号量的值已经是0时，返回EAGAIN错误（不阻塞）
int sem_trywait(sem_t *s);

// 相当于V操作原语
int sem_post(sem_t *s);

// 通过val返回信号量的当前值，当信号量已被占用时，val或为0，或为某个负数，其绝对值为等待该信号量的进程数
int sem_getvalue(sem_t *s, int *val);
```

## 示例程序

### Posix有名信号量示例程序

semdef.h

``` c++
#ifndef __SEMDEF_H__
#define __SEMDEF_H__

#include <unistd.h>
#include <semaphore.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "common.h"

#define SEM_NAME "semt"

#endif
```

semcreate.cpp

``` c++
#include "semdef.h"

int main(int argc, char *argv[])
{
	int semval = 1;
	if (argc > 1)
	{
		semval = atoi(argv[1]);
	}

	if (sem_open(SEM_NAME, O_RDWR|O_CREAT|O_EXCL, 0664, semval) == SEM_FAILED)
	{
		errQuit("open sem failed.");
	}

	printf("create sem %s ok!\n", SEM_NAME);
	exit(0);
}
```

semunlink.cpp

``` c++
#include "semdef.h"

int main(int argc, char *argv[])
{
	if (sem_unlink(SEM_NAME) == -1)
		errQuit("delete failed!");
	printf("delete sem %s ok!\n", SEM_NAME);
	exit(0);
}
```

semgetval.cpp

``` c++
#include "semdef.h"

int main(int argc, char *argv[])
{
	sem_t *s = sem_open(SEM_NAME, 0);
	if (s == SEM_FAILED)
		errQuit("open sem failed.");

	int semval = 0;
	sem_getvalue(s, &semval);
	printf("semval=%d\n", semval);

	sem_close(s);

	exit(0);
}
```

semwait.cpp

``` c++
#include "semdef.h"

void sigInt(int signo)
{
	printf("got SIGINT\n");
	exit(1);
}

int main(int argc, char *argv[])
{
	sem_t *s = sem_open(SEM_NAME, 0);
	if (s == SEM_FAILED)
		errQuit("open sem failed.");

	struct sigaction newAct, oldAct;
	newAct.sa_handler = sigInt;
	sigemptyset(&newAct.sa_mask);
	newAct.sa_flags = 0;

	if (sigaction(SIGINT, &newAct, &newAct) != 0)
	{
		errQuit("setup sigint failed.");
	}

	if (-1 == sem_wait(s))
		errQuit("sem wait failed.");
	printf("P\n");

	sigaction(SIGINT, &oldAct, NULL);

	sem_close(s);
	exit(0);
}
```

sempost.cpp

``` c++
#include "semdef.h"

int main(int argc, char *argv[])
{
	sem_t *s = sem_open(SEM_NAME, 0);
	if (SEM_FAILED == s)
		errQuit("open sem failed.");

	sem_post(s);
	exit(0);
}
```

### 用信号量解决生产者消费者问题

单生产者-单消费者

``` c++
#include <unistd.h>
#include <semaphore.h>
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "common.h"

#define N 16

int gItems = 0;
struct
{
	int buff[N];
	sem_t nEmpty, nStored, mutex;
} gShared;

void* producer(void *arg);
void* consumer(void *arg);

int main(int argc, char *argv[])
{
	if (argc < 2)
	{
		errQuit("please assign items!", false);
	}

	gItems = atoi(argv[1]);
	for (int i = 0; i < N; ++i)
	{
		gShared.buff[i] = -1;
	}

	sem_init(&gShared.nEmpty, 0, N);
	sem_init(&gShared.nStored, 0, 0);
	sem_init(&gShared.mutex, 0, 1);

	pthread_t tidP;
	pthread_t tidC;
	pthread_create(&tidP, NULL, producer, NULL);
	pthread_create(&tidC, NULL, consumer, NULL);

	pthread_join(tidP, NULL);
	pthread_join(tidC, NULL);

	sem_destroy(&gShared.nEmpty);
	sem_destroy(&gShared.nStored);
	sem_destroy(&gShared.mutex);
	exit(0);
}

void* producer(void *arg)
{
	for (int i = 0; i < gItems; ++i)
	{
		sem_wait(&gShared.nEmpty);
		// sem_wait(&gShared.mutex);

		gShared.buff[i % N] = i;

		// sem_post(&gShared.mutex);
		sem_post(&gShared.nStored);
	}
	return NULL;
}

void* consumer(void *arg)
{
	for (int i = 0; i < gItems; ++i)
	{
		sem_wait(&gShared.nStored);
		// sem_wait(&gShared.mutex);

		if (gShared.buff[i % N] != i)
		{
			printf("error! %dth item access conflict, index=%d, curval=%d!\n", i, i%N, gShared.buff[i % N]);
		}

		// sem_post(&gShared.mutex);
		sem_post(&gShared.nEmpty);
	}
	return NULL;
}
```

多生产者-单消费者模型

``` c++
#include <unistd.h>
#include <pthread.h>
#include <semaphore.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "common.h"

#define N 8
#define MAX_THREAD 10

struct Shared
{
	int buff[N];
	int nput;
	int nputval;

	sem_t nEmpty, nStored, mutex;

	Shared()
	{
		memset(buff, -1, sizeof(buff));
		nput = 0;
		nputval = 0;
	}
};

static int gItems = N;
static Shared gShared;

void* producer(void *arg);
void* consumer(void *arg);

int main(int argc, char *argv[])
{
	if (argc > 1)
		gItems = atoi(argv[1]);
	int nProducer = MAX_THREAD;
	if (argc > 2)
		nProducer = min(nProducer, atoi(argv[2]));

	// 初始化信号量
	sem_init(&gShared.nEmpty, 0, N);
	sem_init(&gShared.nStored, 0, 0);
	sem_init(&gShared.mutex, 0, 1);

	// 创建生产者和消费者线程
	pthread_t tidProducer[MAX_THREAD];
	int execTimesOfProducer[MAX_THREAD];
	memset(execTimesOfProducer, 0, sizeof(execTimesOfProducer));
	for (int i = 0; i < nProducer; ++i)
	{
		pthread_create(&tidProducer[i], NULL, producer, &execTimesOfProducer[i]);
	}

	pthread_t tidConsumer;
	pthread_create(&tidConsumer, NULL, consumer, NULL);

	// 等待线程结束
	for (int i = 0; i < nProducer; ++i)
	{
		pthread_join(tidProducer[i], NULL);
	}
	pthread_join(tidConsumer, NULL);

	int totalExecTimes = 0;
	for (int i = 0; i < nProducer; ++i)
	{
		totalExecTimes += execTimesOfProducer[i];
		printf("thread %d exec %d times\n", i, execTimesOfProducer[i]);
	}
	PRINT_INTVAL(totalExecTimes);

	// 销毁信号量
	sem_destroy(&gShared.nEmpty);
	sem_destroy(&gShared.nStored);
	sem_destroy(&gShared.mutex);

	exit(0);
}

void* producer(void *arg)
{
	for (;;)
	{
		sem_wait(&gShared.nEmpty);
		sem_wait(&gShared.mutex);

		if (gShared.nputval >= gItems)
		{
			sem_post(&gShared.nEmpty);
			sem_post(&gShared.mutex);
			return NULL;
		}

		gShared.buff[gShared.nput++%N] = gShared.nputval++;
		(*(int *)arg)++;

		sem_post(&gShared.mutex);
		sem_post(&gShared.nStored);
	}

	return NULL;
}

void* consumer(void *arg)
{
	for (int i = 0; i < gItems; ++i)
	{
		sem_wait(&gShared.nStored);
		if (gShared.buff[i%N] != i)
		{
			printf("conflict! index=%d  val=%d  legal val=%d\n", i%N, gShared.buff[i%N], i);
		}
		sem_post(&gShared.nEmpty);
	}

	return NULL;
}
```

多生产者-多消费者模型

``` c++
#include <unistd.h>
#include <pthread.h>
#include <semaphore.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "common.h"

#define N 16
#define MAX_THREAD 8

struct Shared
{
	int buff[N];
	int nput;
	int nputval;
	int nget;
	int ngetval;

	sem_t nEmpty, nStored, producerMutex, consumerMutex;

	Shared()
	{
		memset(buff, -1, sizeof(buff));
		nput = nputval = 0;
		nget = ngetval = 0;
	}
};

int gItems = 0;
Shared gShared;

void* producer(void *arg);
void* consumer(void *arg);

int main(int argc, char *argv[])
{
	int nProducer = MAX_THREAD;
	int nConsumer = MAX_THREAD;
	gItems = N;

	// 参数设置
	if (argc > 1)
		gItems = atoi(argv[1]);
	if (argc > 2)
		nProducer = atoi(argv[2]);
	if (argc > 3)
		nConsumer = atoi(argv[3]);
	nProducer = min(nProducer, MAX_THREAD);
	nConsumer = min(nConsumer, MAX_THREAD);

	// 信号量初始化
	sem_init(&gShared.nEmpty, 0, N);
	sem_init(&gShared.nStored, 0, 0);
	sem_init(&gShared.producerMutex, 0, 1);
	sem_init(&gShared.consumerMutex, 0, 1);

	// 创建生产者线程
	pthread_t tidProducer[MAX_THREAD];
	int execTimesOfProducer[MAX_THREAD];
	memset(execTimesOfProducer, 0, sizeof(execTimesOfProducer));
	for (int i = 0; i < nProducer; ++i)
	{
		pthread_create(&tidProducer[i], NULL, producer, &execTimesOfProducer[i]);
	}

	// 创建消费者线程
	pthread_t tidConsumer[MAX_THREAD];
	int execTimesOfConsumer[MAX_THREAD];
	memset(execTimesOfConsumer, 0, sizeof(execTimesOfConsumer));
	for (int i = 0; i < nConsumer; ++i)
	{
		pthread_create(&tidConsumer[i], NULL, consumer, &execTimesOfConsumer[i]);
	}

	// 等待线程终止
	for (int i = 0; i < nProducer; ++i)
	{
		pthread_join(tidProducer[i], NULL);
	}
	for (int i = 0; i < nConsumer; ++i)
	{
		pthread_join(tidConsumer[i], NULL);
	}

	// 信号量销毁
	sem_destroy(&gShared.nEmpty);
	sem_destroy(&gShared.nStored);
	sem_destroy(&gShared.producerMutex);
	sem_destroy(&gShared.consumerMutex);

	for (int i = 0; i < nProducer; ++i)
	{
		printf("producer %d run %d times.\n", i, execTimesOfProducer[i]);
	}
	for (int i = 0; i < nConsumer; ++i)
	{
		printf("consumr %d run %d times.\n", i, execTimesOfConsumer[i]);
	}

	exit(0);
}

void* producer(void *arg)
{
	for(;;)
	{
		sem_wait(&gShared.nEmpty);
		sem_wait(&gShared.producerMutex);

		if (gShared.nputval >= gItems)
		{
			sem_post(&gShared.nEmpty);
			sem_post(&gShared.producerMutex);
			return NULL;
		}

		gShared.buff[gShared.nput++%N] = gShared.nputval++;
		(*((int *)arg))++;

		sem_post(&gShared.producerMutex);
		sem_post(&gShared.nStored);
	}
	return NULL;
}

void* consumer(void *arg)
{
	for (;;)
	{
		// 检查是否已消费完
		sem_wait(&gShared.consumerMutex);
		if (gShared.ngetval >= gItems)
		{
			sem_post(&gShared.consumerMutex);
			return NULL;
		}

		// 完成一次消费
		sem_wait(&gShared.nStored);

		if (gShared.buff[gShared.nget%N] != gShared.ngetval)
		{
			printf("confilict! index=%d  curval=%d  legalval=%d\n",
				   gShared.nget%N, gShared.buff[gShared.nget%N], gShared.nget);
		}

		gShared.nget++;
		gShared.ngetval++;
		(*((int *)arg))++;

		sem_post(&gShared.nEmpty);
		sem_post(&gShared.consumerMutex);
	}
	return NULL;
}
```

### 多缓冲区方案

多缓冲区示例(单生产者-单消费者)

``` c++
#include <unistd.h>
#include <pthread.h>
#include <semaphore.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "common.h"

#define BUFF_SIZE 256
#define QSIZE 8

struct Shared
{
     struct
     {
		 char buff[BUFF_SIZE];
		 int n;
     } que[QSIZE];

	sem_t nEmpty, nStored;

	Shared()
	{
		memset(que, 0, sizeof(que));
	}
};
struct Shared gShared;

void* producer(void *arg);
void* consumer(void *arg);

int main(int argc, char *argv[])
{
	sem_init(&gShared.nEmpty, 0, QSIZE);
	sem_init(&gShared.nStored, 0, 0);

	pthread_t tidProducer;
	int execTimesOfProducer = 0;
	pthread_create(&tidProducer, NULL, producer, &execTimesOfProducer);

	pthread_t tidConsumer;
	int execTimesOfConsumer = 0;
	pthread_create(&tidConsumer, NULL, consumer, &execTimesOfConsumer);

	pthread_join(tidProducer, NULL);
	pthread_join(tidConsumer, NULL);

	printf("producer exec %d times\n", execTimesOfProducer);
	printf("consumer exec %d times\n", execTimesOfConsumer);

	sem_destroy(&gShared.nEmpty);
	sem_destroy(&gShared.nStored);

	exit(0);
}

void* producer(void *arg)
{
	int i = 0;
	for (;;)
	{
		sem_wait(&gShared.nEmpty);

		gShared.que[i].n = read(STDIN_FILENO, gShared.que[i].buff, BUFF_SIZE);
		if (gShared.que[i].n < 0)
		{
			errQuit("read error!");
		}
		if (gShared.que[i].n == 0)
		{
			sem_post(&gShared.nStored);
			return NULL;
		}

		i = (i+1)%QSIZE;
		(*((int *)arg))++;

		sem_post(&gShared.nStored);
	}
	return NULL;
}

void* consumer(void *arg)
{
	int i = 0;
	for (;;)
	{
		sem_wait(&gShared.nStored);

		if (0 == gShared.que[i].n)
		{
			return NULL;
		}
		if (write(STDOUT_FILENO, gShared.que[i].buff, gShared.que[i].n) != gShared.que[i].n)
		{
			errQuit("write error!");
		}
		i = (i+1)%QSIZE;
		(*((int *)arg))++;

		sem_post(&gShared.nEmpty);
	}
	return NULL;
}
```
