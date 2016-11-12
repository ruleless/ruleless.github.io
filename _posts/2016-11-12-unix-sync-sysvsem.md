---
layout: post
title: "System V信号量"
description: ""
category: Unix环境编程
tags: [unix同步机制]
---
{% include JB/setup %}

## 概要

信号量按其取值可分为两种：

  1. 二值信号量：其值为0或1的信号量。这与互斥锁类似，若资源被锁住则信号量值为0，若资源可用则信号量值为1；
  2. 计数信号量：其值在0和某个限制值（对于Posix信号量，该值必须至少为32767）之间的信号量。
     我们使用这些信号量在生产者-消费者问题中记录可用资源数，信号量的值就是可用资源数。


**计数信号量集**：一个或多个计数信号量构成计数信号量集。
System V即通过提供操作计数信号量集的接口来实现信号量。
当我们谈论"System V信号量"时，所指的是计数信号量集；
当我们谈论"Posix信号量"时，所指的是单个计数信号量。

对于系统中的每个信号量集，内核维护一个semid_ds结构：

``` c++
#include <sys/sem.h>

struct semid_ds
{
	struct ipc_perm sem_perm; /* operation permission struct */
	unsigned long int sem_nsems; /* number of semaphores in set */
	__time_t sem_ctime; /* last time changed by semctl() */
	__time_t sem_otime; /* last semop() time */
	struct sem *sem_base; // 应用程序不可见
	unsigned long int __unused1;
	unsigned long int __unused2;
	unsigned long int __unused3;
	unsigned long int __unused4;
};

// sem结构是内核用于维护某个给定信号量的一组值的内部数据结构，一般至少包含下列成员
struct sem
{
	ushort_t semval; // semaphore value, nonnegative
	short sempid; // PID of last successful semop(), SETVAL, SETALL
	ushort_t semncnt; // awaiting semval > current value
	ushort_t semzcnt; // awaiting semval = 0
};
```

## 信号量创建、获取函数

``` c++
#include <sys/sem.h>

// 成功则返回非负标识符，出错返回-1
int semget(key_t key, int nsems, int oflag);
```

nsems参数指定集合中的信号量个数，如果我们不是创建一个新的信号集而只是访问一个已存在的集合，那就可以把该参数指定为0。
一旦创建完一个信号量集合，我们就不能再改变其中的信号量数目。

oflag参数是访问权限位（一般为0644）和IPC_CREAT、IPC_EXCL的组合。

当实际操作为创建一个新的信号量集时，相应的semid_ds结构成员初始化如下：

  1. sem_perm结构的uid和cuid被初始化为进程的有效用户ID，gid和cgid被初始化为进程的有效组ID
  2. oflag参数中的访问权限位存入sem_perm.mode
  3. sem_otime被初始化为0，sem_ctime则被初始化为当前时间
  4. sem_nsems被初始化为参数nsems的值
  5. 与该集合中每个信号量关联的sem结构并不初始化，这些结构是在以SETVAL或SETALL命令调用semctl时初始化的

## 信号量PV操作

``` c++
#include <sys/sem.h>

int semop(int semid, struct sembuf *ops, int opcnt); // 成功返回0，失败返回-1

struct sembuf // 操作参数
{
	unsigned short int sem_num;   /* semaphore number */
	short int sem_op;     /* semaphore operation */
	short int sem_flg;        /* operation flag */
};
```

为讨论方便，先作如下说明：

  * semval：表示信号量的当前值
  * semncnt：等待semval>=所请求资源数的线程数
  * semzcnt：等待semval=0的线程数
  * semadj：所指定信号量针对调用进程的调整值。只有在对应本操作的sembuf.sem_flg指定SEM_UNDO标志后，semadj才会更新。
    这是一个概念性的变量，它由内核为其在某个信号量操作中指定了SEM_UNDO标志的各个进程维护，不必存在名为semadj的结构成员。
  * 以非阻塞方式操作一个给定信号量的方法是，在对应的sembuf结构的sem_flg成员中指定IPC_NOWAIT标志。
  * semop是可被信号中断的慢速系统调用，当引起睡眠的semop函数被信号中断时，将返回EINTR。
  * 当一个线程被投入睡眠以等待某个信号量操作完成时，如果该信号量被另外某个线程或进程从系统中删除，
    那么引起睡眠的semop函数将返回EIDRM错误。


现在，我们来讨论System V信号量的PV操作：

  1. 如果sem_op是正数，其值就加到semval上，这相当于V操作；如果指定了SEM_UNDO标志，
     那么就从相应信号量的semadj值中减去sem_op

  2. 如果sem_op是负数，此时的语义可理解为调用者申请sem_op个资源，这相当于P操作

     1. 如果semval>=\|sem_op\|，那么执行semval+=sem_op；如果指定了SEM_UNDO标志，那么执行semad-=semadj
	 2. 如果semval<\|sem_op\|，那么相应的++ semncnt，调用线程被阻塞到semval>=\|sem_op\|之时，到那时也会执行--semncnt；
	    如果指定了IPC_NOWAIT标志，semop将立即返回EAGAIN错误

  3. 如果sem_op等于0，那么调用者希望semval变为0，如果semval==0，那么立即返回，
     如果semval!=0，相应的semzcnt就加1，等到semval==0之时semop返回，semzcnt减1；
	 如果指定了IPC_NOWAIT标志，并且semval!=0，semop立即返回EAGAIN错误

## 信号量控制

``` c++
#include <sys/sem.h>

// 成功返回非负值（与命令相关），失败返回-1
int semctl(int semid, int semnum, int cmd, ... /* union semun arg */);
```

semnum标识信号量集内的某个成员，其仅用于GETVAL、SETVAL、GETNCNT、GETZCNT、GETPID等命令

第四个参数是可选的，取决于cmd

``` c++
union semun {
	int val;            /* value for SETVAL */
	struct semid_ds *buf;   /* buffer for IPC_STAT & IPC_SET */
	unsigned short *array;  /* array for GETALL & SETALL */
	struct seminfo *__buf;  /* buffer for IPC_INFO */
	void *__pad;
};
```

这个联合对应用程序是不可见的，所以我们必须在应用程序中声明它
cmd命令定义如下：

  * **GETVAL** 返回信号量的当前值
  * **SETVAL** 设置信号量的当前值
  * **GETPID** PID of last successful semop(), SETVAL, SETALL
  * **GETNCNT** awaiting semval > current value
  * **GETZCNT** awaiting semval = 0
  * **GETALL** 通过semun.array返回信号量集中所有的信号量的当前值
  * **SETALL** 设置信号量集中每个信号量的当前值
  * **IPC_RMID** 删除信号量集
  * **IPC_SET** 设置跟信号量集关联的semid_ds结构中的：sem_perm.uid、sem_perm.gid、sem_perm.mode
  * **IPC_STAT** 通过semun.buf参数返回信号量集当前的semid_ds结构

## 示例程序

semdef.h

``` c++
#ifndef __SEMDEF_H__
#define __SEMDEF_H__

#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/sem.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define SEM_PATH "./semcreate"

union semun
{
	int val;
	struct semid_ds *buf;
	unsigned short *array;
};

extern void errQuit();

#endif
```

semdef.cpp

``` c++
#include "semdef.h"

void errQuit()
{
	printf("%s\n", strerror(errno));
	exit(1);
}
```

semcreate.cpp

``` c++
#include "semdef.h"

int main(int argc, char *argv[])
{
	key_t key = ftok(SEM_PATH, 1);
	if (key < 0)
		errQuit();

	int semnum = 1;
	if (argc > 1)
		semnum = atoi(argv[1]);

	if (semget(key, semnum, 0644|IPC_CREAT|IPC_EXCL) < 0)
		errQuit();

	printf("create sem ok!\n");

	exit(0);
}
```

semsetvals.cpp

``` c++
#include "semdef.h"

int main(int argc, char *argv[])
{
	key_t key = ftok(SEM_PATH, 1);
	if (key < 0)
		errQuit();

	int semid = semget(key, 0, 0);
	if (semid < 0)
		errQuit();

	struct semid_ds ds;
	union semun arg;
	arg.buf = &ds;
	if (semctl(semid, 0, IPC_STAT, arg) < 0)
		errQuit();

	unsigned short *semarr = new unsigned short[ds.sem_nsems];
	memset(semarr, 0, sizeof(unsigned short)*ds.sem_nsems);
	for (int i = 1; i < argc && i-1 < ds.sem_nsems; ++i)
		semarr[i-1] = atoi(argv[i]);

	arg.array = semarr;
	if (semctl(semid, 0, SETALL, arg) < 0)
		errQuit();

	exit(0);
}
```

semgetvals.cpp

``` c++
#include "semdef.h"

int main(int argc, char *argv[])
{
	key_t key = ftok(SEM_PATH, 1);
	if (key < 0)
		errQuit();

	int semid = semget(key, 0, 0);
	if (semid < 0)
		errQuit();

	struct semid_ds ds;
	union semun arg;
	arg.buf = &ds;
	if (semctl(semid, 0, IPC_STAT, arg) < 0)
		errQuit();

	unsigned short *semarr = new unsigned short[ds.sem_nsems];
	arg.array = semarr;
	if (semctl(semid, 0, GETALL, arg) < 0)
		errQuit();

	for (int i = 0; i < ds.sem_nsems; ++i)
		printf("sem %d: %d\n", i, semarr[i]);

	exit(0);
}
```

semrm.cpp

``` c++
#include "semdef.h"

int main(int argc, char *argv[])
{
	key_t key = ftok(SEM_PATH, 1);
	if (key < 0)
		errQuit();

	int semid = semget(key, 0, 0);
	if (semid < 0)
		errQuit();

	if (semctl(semid, 0, IPC_RMID) == 0)
		printf("rm ok.\n");

	exit(0);
}
```

semop.cpp

``` c++
#include "semdef.h"

int main(int argc, char *argv[])
{
	key_t key = ftok(SEM_PATH, 1);
	if (key < 0)
		errQuit();

	int semid = semget(key, 0, 0);
	if (semid < 0)
		errQuit();

	struct semid_ds ds;
	union semun arg;
	arg.buf = &ds;
	if (semctl(semid, 0, SEM_STAT, arg) < 0)
		errQuit();

	static const int N = 10;
	struct sembuf opBufs[N];
	int n = 0;
	for (int i = 1; i < argc; ++i)
	{
		opBufs[n].sem_op = atoi(argv[i]);
		opBufs[n].sem_num = i-1;
		opBufs[n].sem_flg = 0;
		++n;
	}
	if (semop(semid, opBufs, n) < 0)
		errQuit();

	exit(0);
}
```
