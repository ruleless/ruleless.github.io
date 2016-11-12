---
layout: post
title: "Posix 共享内存区"
description: ""
category: Unix环境编程
tags: [unix ipc]
---
{% include JB/setup %}

共享内存区是可用IPC形式中最快的。

## 概要

使用共享内存区有两个步骤：

  1. 创建或打开底层支撑对象：文件或共享内存区对象。
  2. 内存映射：创建用户进程与底层支撑对象之间的交互通道。

根据底层支撑对象的不同，有两种形式的Posix共享内存区IPC：

  1. 使用Posix内存映射文件
  2. 使用Posix共享内存区对象

## 内存映射

``` c++
#include <sys/mman.h>

void* mmap(void *addr, size_t len, int prot, int flags, int fd, off_t offset);
```

mmap函数把一个文件或一个Posix共享内存区对象映射到调用进程的地址空间。使用该函数一般出于下述目的：

  + 使用普通文件以提供内存映射I/O
  + 使用特殊文件以提供匿名内存映射
  + 使用shm_open以提供无亲缘关系进程间的Posix共享内存区

mmap参数说明：

  + `addr`指定fd所对应的文件应被映射到的进程内空间的起始地址，通常为空。
	表示由内核自己去选择起始地址。

  + `len` 映射到调用进程地址空间的字节数

  + `prot` 指定内存映射区的访问权限（常指定为PROT_READ\|PROT_WRITE）。

    - `PROT_READ` 数据可读
	- `PROT_WRITE` 数据可写
	- `PROT_EXEC` 数据可执行
	- `PROT_NONE` 数据不可访问

  + `flags` MAP_SHARED或MAP_PRIVATE必须指定其中一个，并可有选择地或上MAP_FIXED

    - `MAP_PRIVATE` 调用进程对内存映射区所作的修改只对该进程可见，不改变底层支撑对象
	- `MAP_SHARED` 调用进程对内存映射区所作的修改对所有共享底层支撑对象的进程可见，并且映射区的数据更改也会反映到底层支撑对象
	- `MAP_FIXED` 从移植性上考虑，MAP_FIXED不应该指定。如果没有指定该标志，但addr不是一个空指针，那么addr如何处置取决于实现；
	  可移植的代码应把addr指定成一个空指针，并且不指定MAP_FIXED

  + `fd` 打开的文件描述符

  + `offset` 文件被映射的偏移位置

``` c++
#include <sys/mman.h>

int munmap(void *addr, int len); // 成功返回0，失败返回-1
```

munmap可从进程的地址空间删除一个映射关系。其中addr参数是由mamp返回的地址，len是映射区的大小。
再次访问这些地址将导致产生SIGSEGV信号。

## Posix内存映射I/O

将打开着的普通文件或特殊文件的文件描述符传递给mmap便可将文件中的一块区域映射到调用进程的地址空间，
我们可以直接通过读写内存的方式访问文件，这便是内存映射I/O的含义。
调用mmap将文件映射到调用进程地址空间的示意图如下：

![](/images/unix/ipc/posix-shm-io.png)

在指定MAP_SHARED标志的前提下，内核的虚拟内存算法保持内存映射文件与内存映射区的同步。
这种同步并不一定是实时的，幸运的是，我们可以调用msync来强制执行这种同步以保证内存映射文件与内存映射区中数据的一致性。

``` c++
#include <sys/mman.h>

int msync(void *addr, size_t len, int flags); // 成功返回0，失败返回-1
```

flags取值如下：

  + MS_ASYNC 执行异步写
  + MS_SYNC 执行同步写
  + MS_INVALIDATE 使高速缓存的数据失效

## Posix共享内存区

使用Posix共享内存区需要两个步骤：

  1. 指定一个名字参数调用shm_open，以创建一个新的共享内存区对象或打开一个已存在的共享内存区对象；
  2. 调用mmap把这个共享内存区对象映射到调用进程的地址空间。

传递给shm_open的名字参数随后由希望共享该内存区的任何其他进程使用。

``` c++
#include <sys/mman.h>

// 成功则返回非负描述符，失败则返回-1
int shm_open(const char *name, int oflag, mode_t mode);

// 成功返回0，失败返回-1
int shm_unlink(const char *name);
```

通过shm_open创建的共享内存区可以使用ftruncate来调整其大小：

``` c++
#include <unistd.h>

int ftruncate(int fd, off_t length); // 成功返回0，失败返回-1
```

针对一个已打开的共享内存区对象，我们可以使用fsate获取有关该对象的信息：

``` c++
#include <sys/types.h>
#include <sys/stat.h>

int fstat(int fd, struct stat *buf); // 成功返回0，失败返回-1
```

当fd指代一个共享内存区对象时，struct stat只有四个成员有效：

``` c++
struct stat
{
	...
	mode_t st_mode;
	uid_t st_uid;
	gid_t st_gid;
	off_t st_size;
};
```

## 示例程序

shmdef.h

``` c++
#ifndef __SHMDEF_H__
#define __SHMDEF_H__

#include <unistd.h>
#include <fcntl.h>
#include <semaphore.h>
#include <sys/mman.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "common.h"

#define SHMNAME "/shmtest"

#define N 256

#endif
```

shmio.cpp(内存映射I/O示例)

``` c++
#include "shmdef.h"

#define MAX_SIZE 8196

int main(int argc, char *argv[])
{
	const char *fileName = "tmp";
	if (argc > 1)
		fileName = argv[1];

	int fd = open(fileName, O_RDWR|O_CREAT, 0644);
	if (fd < 0)
		errQuit("open file failed.");

	void *ptr = mmap(NULL, MAX_SIZE, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
	ftruncate(fd, 0);

	int n = 0;
	int cur = 0;
	char buff[MAX_SIZE];
	while ((n=read(STDIN_FILENO, buff, MAX_SIZE)) > 0)
	{
		n = min(n, MAX_SIZE-cur);
		ftruncate(fd, cur+n);
		memcpy(ptr+cur, buff, n);
		cur += n;

		if (cur >= MAX_SIZE)
			break;
	}

	exit(0);
}
```

shmcreate.cpp

``` c++
#include "shmdef.h"

int main(int argc, char *argv[])
{
	int fd = shm_open(SHMNAME, O_RDWR|O_CREAT|O_EXCL, 0644);
	if (fd < 0)
		errQuit("open shm failed.");
	printf("create shm sucess.\n");
	exit(0);
}
```

shmunlink.cpp

``` c++
#include "shmdef.h"

int main(int argc, char *argv[])
{
	shm_unlink(SHMNAME);
	exit(0);
}
```

Makefile

``` shell
SNAIL_ROOT:=../../../snail
SNAIL_INC:=$(SNAIL_ROOT)/include
SNAIL_LIB:=$(SNAIL_ROOT)/lib

LIB_DIR:=$(SNAIL_ROOT)/lib

CXXFLAGS:=-g -Wall -I $(subst :, -I,$(SNAIL_INC))
LDFLAGS:=-L $(LIB_DIR) -lsnail -lrt

.PHONY:=lib all clean

all:lib bin

lib:
     cd $(SNAIL_ROOT) && $(MAKE)
bin:shmcreate shmunlink shmio

shmcreate:shmcreate.o
     $(CXX) $^ $(LDFLAGS) -o $@
shmunlink:shmunlink.o
     $(CXX) $^ $(LDFLAGS) -o $@
shmio:shmio.o
     $(CXX) $^ $(LDFLAGS) -o $@

shmcreate.o:shmcreate.cpp shmdef.h
shmunlink.o:shmunlink.cpp shmdef.h
shmio.o:shmio.cpp shmdef.h

clean:
     -rm *.o shmcreate shmunlink shmio
```

## 基于生产者-消费者模型的回射程序

shmdef.h

``` c++
#ifndef __SHMDEF_H__
#define __SHMDEF_H__

#include <unistd.h>
#include <semaphore.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define QSize 1
#define MSGSize 256

#define SHM_NAME "shmecho"

struct Shared
{
     char msg[QSize][MSGSize];
     int put;
     sem_t nEmpty;
     sem_t nStored;
     sem_t mutex;

     int nOverflow;
     sem_t overflowMutex;

     Shared()
     {
          memset(msg, 0, sizeof(msg));
          put = 0;
          nOverflow = 0;
     }
};

#endif
```

server.cpp

``` c++
#include <pthread.h>
#include <sys/mman.h>
#include "common.h"
#include "shmdef.h"

int main(int argc, char *argv[])
{
     // 创建Posix共享内存区
     shm_unlink(SHM_NAME);
     int shmFd = shm_open(SHM_NAME, O_RDWR|O_CREAT|O_EXCL, 0664);
     if (shmFd < 0)
     {
          errQuit("create shm failed.");
     }
     ftruncate(shmFd, sizeof(Shared));
     Shared *ptr = (Shared *)mmap(NULL, sizeof(Shared), PROT_READ|PROT_WRITE, MAP_SHARED, shmFd, 0);
     if (MAP_FAILED == ptr)
     {
          errQuit("mmap failed.", false);
     }
     close(shmFd);

     // 初始化信号量
     sem_init(&ptr->nEmpty, 1, QSize);
     sem_init(&ptr->nStored, 1, 0);
     sem_init(&ptr->mutex, 1, 1);
     sem_init(&ptr->overflowMutex, 1, 1);

     // 消费
     int index = 0;
     int lastOverflow = 0;
     for (;;)
     {
          sem_wait(&ptr->nStored);
          printf("%s\n", ptr->msg[index]);
          index = (index + 1) % QSize;
          sem_post(&ptr->nEmpty);

          sem_wait(&ptr->overflowMutex);
          if (ptr->nOverflow != lastOverflow)
          {
               printf("overflow:%d\n", ptr->nOverflow);
               lastOverflow = ptr->nOverflow;
          }
          sem_post(&ptr->overflowMutex);
     }

     // 销毁信号量
     sem_destroy(&ptr->nEmpty);
     sem_destroy(&ptr->nStored);
     sem_destroy(&ptr->mutex);

     exit(0);
}
```

client.cpp

``` c++
#include <pthread.h>
#include <sys/mman.h>
#include "shmdef.h"
#include "common.h"

int main(int argc, char *argv[])
{
     // 打开共享内存区
     int shmFd = shm_open(SHM_NAME, O_RDWR, 0);
     if (shmFd < 0)
     {
          errQuit("open shm failed.");
     }
     Shared *ptr = (Shared *)mmap(NULL, sizeof(Shared), PROT_READ|PROT_WRITE, MAP_SHARED, shmFd, 0);
     if (MAP_FAILED == ptr)
     {
          errQuit("mmap failed.", false);
     }
     close(shmFd);

     // 生产
     char buff[MSGSize] = {0};
     while (fgets(buff, MSGSize, stdin))
     {
          if (-1 == sem_wait(&ptr->nEmpty))
          {
               sem_wait(&ptr->overflowMutex);
               ++ptr->nOverflow;
               sem_post(&ptr->overflowMutex);
               continue;
          }

          sem_wait(&ptr->mutex);

          int n = strlen(buff);
          if (n > 0 && buff[n-1] == '\n')
               buff[n-1] = 0;
          snprintf(ptr->msg[ptr->put], MSGSize, "client %d say(bufidx=%d): %s", (int)getpid(), ptr->put, buff);
          ptr->put = (ptr->put + 1) % QSize;

          sem_post(&ptr->mutex);
          sem_post(&ptr->nStored);
     }

     exit(0);
}
```

Makefile

``` shell
commdir:=../../../common
incdir:=$(commdir)

VPATH:=$(incdir)

cc:=g++
CXXFLAGS:=-g -Wall -lpthread -lrt -I $(subst :, -I,$(incdir))

.PHONY:all clean
all:client server

client:client.o common.o
     $(cc) -o client $^ $(CXXFLAGS)
server:server.o common.o
     $(cc) -o server $^ $(CXXFLAGS)

client.o:client.cpp shmdef.h
server.o:server.cpp shmdef.h
common.o:common.cpp

clean:
     -rm *.o client server
```
