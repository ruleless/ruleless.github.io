---
layout: post
title: "System V 共享内存区"
description: ""
category: Unix环境编程[进程IPC]
tags: [进程IPC]
---
{% include JB/setup %}

System V共享内存区在概念上类似于Posix共享内存区。
代之以调用shm_open后调用mmap的是：先调用shmget，再调用shmat。
对于每个共享内存区，内核维护如下的信息结构：

``` c++
#include <sys/shm.h>

struct shmid_ds
{
	struct ipc_perm     shm_perm;      /* operation perms */
	int                 shm_segsz;     /* size of segment (bytes) */
	__kernel_time_t     shm_atime;     /* last attach time */
	__kernel_time_t     shm_dtime;     /* last detach time */
	__kernel_time_t     shm_ctime;     /* last change time */
	__kernel_ipc_pid_t  shm_cpid;      /* pid of creator */
	__kernel_ipc_pid_t  shm_lpid;      /* pid of last operator */
	unsigned short      shm_nattch;    /* no. of current attaches */
	unsigned short      shm_unused;    /* compatibility */
	void               *shm_unused2;   /* ditto - used by DIPC */
	void               *shm_unused3;   /* unused */
};
```

## 创建函数

``` c++
#include <sys/shm.h>

// 若成功则返回共享内存区对象标识符，失败返回-1
int shmget(key_t key, size_t size, int oflag);
```

shmget创建一个新的共享内存区或访问一个已存在的共享内存区。

oflag指定了IPC对象的打开状态标志和访问权限。
System V共享内存区的打开状态标志可为：IPC_CREAT、IPC_EXCL。
访问权限一般指定为0644(rw-r--r--)。

## 控制函数

``` c++
#include <sys/shm.h>

// 成功返回0，失败返回-1
int shmctl(int shmid, int cmd, struct shmid_ds *buf);
```

shmctl提供了对一个共享内存区的多种操作方式：

  + `IPC_RMID` 从系统中删除由shmid标识指代的共享内存区
  + `IPC_SET` 给所指定的共享内存区设置其shmid_ds结构的以下三个成员：shm_perm.uid、shm_perm.gid、shm_perm.mode。
  + `IPC_STAT` 返回共享内存区的当前shmid_ds结构

## 内存附接、断接

``` c++
#include <sys/shm.h>

void* shmat(int shmid, const void *addr, int flag); // 成功则返回映射区的起始地址，失败返回-1
int shmdt(const void *addr); // 成功返回0，失败返回-1
```

shmat返回的映射区地址由以下规则确定：

  + 如果参数addr是一个空指针，那么由系统替调用者选择地址。这也是可移植性最好的方法；
  + 如果参数addr是一个非空指针，那么返回地址取决于调用者是否给flag参数指定了SHM_RND值：
    - 如果没有指定SHM_RND，那么相应的共享区附接到由addr参数指定的地址；
	- 如果指定了SHM_RND，那么相应的共享内存区附接到由addr参数指定的地址向下舍入一个SHMLBA常值。
	  LBA代表“低端边界地址(lower boundary address)”。


默认情况下，只要调用进程具有某个共享内存区的读写权限，它附接到该内存区后就能够同时读写该内存区。
flag参数中也可以指定SHM_RDONLY常值来限定只读访问。

## 示例程序

shmdef.h

``` c++
#ifndef __SHMDEF_H__
#define __SHMDEF_H__

#include <unistd.h>
#include <sys/shm.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define SHM_PATH "shmget"
#define MY_SHM_ID 1

#define SHM_SIZE 1024

#endif
```

shmget.cpp

``` c++
#include "shmdef.h"
#include "common/common.h"

int main(int argc, char *argv[])
{
	int size = SHM_SIZE;
	if (argc > 1)
		size = atoi(argv[1]);

	key_t key = ftok(SHM_PATH, MY_SHM_ID);
	if (key < 0)
		errQuit("ftok err.");

	if (shmget(key, size, IPC_CREAT|IPC_EXCL|0644) < 0)
		errQuit("create shm failed.");

	printf("create shm success.\n");
	exit(0);
}
```

shmrm.cpp

``` c++
#include "shmdef.h"
#include "common/common.h"

int main(int argc, char *argv[])
{
	key_t key = ftok(SHM_PATH, MY_SHM_ID);
	if (key < 0)
		errQuit("ftok err");

	int shmid = shmget(key, 0, 0);
	if (shmid < 0)
		errQuit("get shm failed.");

	if (shmctl(shmid, IPC_RMID, NULL) < 0)
		errQuit("rm shm failed.");

	printf("rm shm ok!\n");
	exit(0);
}
```

shmwrite.cpp

``` c++
#include "shmdef.h"
#include "common/common.h"

#define BUFF_SIZE 512

int main(int argc, char *argv[])
{
	key_t key = ftok(SHM_PATH, MY_SHM_ID);
	if (key < 0)
		errQuit("ftok err.");

	int shmid = shmget(key, 0, 0);
	if (shmid < 0)
		errQuit("open shm failed.");

	char *ptr = (char *)shmat(shmid, NULL, 0);
	if (ptr < 0)
		errQuit("attach shm err.");

	struct shmid_ds shmds;
	if (shmctl(shmid, IPC_STAT, &shmds) < 0)
		errQuit("get shm stat err.");

	int shmSize = shmds.shm_segsz;
	int cur = 0;
	char buff[BUFF_SIZE];
	int n = 0;
	while ((n = read(STDIN_FILENO, buff, BUFF_SIZE)) > 0)
	{
		int cpLen = min(n, shmSize-cur);
		if (cpLen > 0)
		{
			memcpy(ptr+cur, buff, cpLen);
			cur += cpLen;

			if (cur >= shmSize)
				break;
		}
		else
		{
			break;
		}
	}
	exit(0);
}
```

shmread.cpp

``` c++
#include "shmdef.h"
#include "common/common.h"

int main(int argc, char *argv[])
{
	key_t key = ftok(SHM_PATH, MY_SHM_ID);
	if (key < 0)
		errQuit("ftok err.");

	int shmid = shmget(key, 0, 0);
	if (shmid < 0)
		errQuit("open shm err.");

	void *ptr = shmat(shmid, NULL, 0);
	if (ptr < 0)
		errQuit("attach shm err.");

	struct shmid_ds shmds;
	shmctl(shmid, IPC_STAT, &shmds);
	write(STDOUT_FILENO, ptr, shmds.shm_segsz);

	exit(0);
}
```
