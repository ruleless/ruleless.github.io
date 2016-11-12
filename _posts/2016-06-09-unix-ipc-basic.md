---
layout: post
title: "进程IPC概要"
description: ""
category: Unix环境编程
tags: [unix ipc]
---
{% include JB/setup %}

## Unix进程间的数据共享

![](/images/unix/ipc/ipc-data-sharing.png)

Unix进程间的数据共享有三种方式：

  1. 左边的两个进程通过访问文件的方式共享数据，这通常需要某种形式的同步，一般用文件记录锁。

  2. 中间的两个进程共享驻留于内核中的某些数据。
     **管道、FIFO、System V消息队列、System V信号量** 属于这类型的数据共享方式；
	 **Posix消息队列、Posix信号量** 在某些系统的实现也属于这种。

  3. 右边的两个进程有一个双方都能访问的共享内存区，若该共享内存区被映射到了各自进程的地址空间，那么这两个进程将能不通过内核（系统调用）而实现交互。
     共享该内存区的进程需要某种形式的同步，信号量是常用的选择。

## IPC对象的持续性

IPC对象的持续性是指IPC对象存在的时长，有三种类型的持续性：


  1. 随进程的持续性：此类IPC对象一直存在到该对象的最后一个进程关闭该对象为止。**管道、FIFO** 属此类

  2. 随内核的持续性：此类IPC对象一直存在到内核重新自举或显示删除该对象为止。
     **System V消息队列、信号量、共享内存区** 属此类；
	 **Posix消息队列、信号量、共享内存区** 必须至少是随内核持续的，
	 但也可以是随文件系统持续的，具体取决于实现

  3. 随文件系统的持续性：此类IPC对象一直存在到显示删除该对象为止。
     **Posix消息队列、信号量、共享内存区** 如果是使用内存映射文件实现的，那么它就是随文件系统持续的

## 进程间数据交互方式汇总

![](/images/unix/ipc/ipc-data-sharing-sum.png)

## 名字空间

对于一种给定的IPC类型，其可能的名字的集合称为它的名字空间。
名字空间非常重要，因为对于除普通管道以外的所有形式的IPC来说，
名字是客户与服务端彼此连接以交换消息的手段。

  + **管道**

    + 名字：没有名字
	+ 打开后的标识：描述符

  + **FIFO**

	+ 名字：路径名
    + 打开后的标识：描述符

  + **Posix消息队列**

	+ 名字：Posix IPC名字
	+ 打开后的标识：mqd_t

  + **Posix有名信号量**

	+ 名字：Posix IPC名字
	+ 打开后的标识：sem_t

  + **Posix基于内存的信号量**

    + 名字：没有名字
	+ 打开后的标识：sem_t

  + **Posix共享内存区**

    + 名字：Posix IPC名字
	+ 打开后的标识：描述符

  + **System V消息队列**

    + 名字：key_t键
	+ 打开后的标识：System V IPV标识符

  + **System V信号量**

    + 名字：key_t键
	+ 打开后的标识：System V IPV标识符

  + **System V共享内存区**

    + 名字：key_t键
	+ 打开后的标识：System V IPV标识符

  + **Posix互斥锁**

    + 名字：没有名字
	+ 打开后的标识：pthread_mutex_t指针

  + **Posix条件变量**

    + 名字：没有名字
    + 打开后的标识：pthread_cond_t指针

  + **Posix读写锁**

	+ 名字：没有名字
    + 打开后的标识：pthread_rwlock_t指针

  + **fncl记录锁**

    + 名字：路径名
	+ 打开后的标识：描述符

## Posix IPC

**Posix消息队列、Posix信号量、Posix共享内存区** 合称“Posix IPC”。

### Posix IPC函数汇总

  + Posix消息队列

    + 头文件：`<mqueue.h>`
    + IPC创建、销毁函数：`mq_open/mq_close/mq_unlink`
	+ IPC控制函数：`mq_getattr/mq_setattr`
	+ IPC操作函数：`mq_receive/mq_send/mq_notify`

  + Posix信号量

    + 头文件：`<semaphore.h>`
    + IPC创建、销毁函数：`sem_open/sem_close/sem_unlink/sem_init/sem_destroy`
	+ IPC控制函数：无
	+ IPC操作函数：`sem_wait/sem_trywait/sem_post/sem_getvalue`

  + Posix共享内存区

    + 头文件：`<sys/mman.h>`
    + IPC创建、销毁函数：`shm_open/shm_unlink`
	+ IPC控制函数：`ftruncate/fstat`
	+ IPC操作函数：`mmap/munmap`

### Posix IPC名字

Posix.1规定的IPC名字规范：

  1. 它必须符合已有的路径名规则（必须最多由PATH_MAX个字节组成，包括结尾的空字节）
  2. 如果它以/开头，那么对IPC函数的不同调用将访问同一个队列。如果它不以/开头，那么效果取决于实现
  3. 名字中额外的/的解释由实现定义

当我们指定一个只有/作为首字符的名字时，
在某些系统上(如Digital Unix)，我们必须在根目录下具有写权限才能创建IPC对象；
而当我们指定具有写权限的路径名时(例如：/tmp/que.123)，某些Unix(例如Solaris)则创建失败。

### 创建与打开IPC通道

mq_open、sem_open、shm_open这三个函数创建或打开一个IPC对象，它们的第二个参数指定怎样打开所请求的对象。
这与open函数的第二个参数类似。下面总结了各打开方式针对IPC对象打开函数的有效性：

  + mq_open

    + 只读 **O_RDONLY**
	+ 只写 **O_WRONLY**
	+ 读写 **O_RDWR**
	+ 若不存在则创建 **O_CREAT**
	+ 排他性创建 **O_EXCL**
	+ 非阻塞模式 **O_NONBLOCK**
	+ 若已存在则截短 **无效**

  + sem_open

    + 只读 **无效**
	+ 只写 **无效**
	+ 读写 **无效**
	+ 若不存在则创建 **O_CREAT**
	+ 排他性创建 **O_EXCL**
	+ 非阻塞模式 **无效**
	+ 若已存在则截短 **无效**

  + shm_open

    + 只读 **O_RDONLY**
	+ 只写 **无效**
	+ 读写 **O_RDWR**
	+ 若不存在则创建 **O_CREAT**
	+ 排他性创建 **O_EXCL**
	+ 非阻塞模式 **无效**
	+ 若已存在则截短 **O_TRUNC**


## System V IPC

**System V消息队列、System V信号量、System V共享内存区** 合称 System V IPC。

### System V IPC函数汇总

  + 消息队列

    + 头文件： `<sys/msg.h>`
    + IPC对象创建或打开函数： `msgget`
	+ IPC控制函数： `msgctl`
	+ IPC操作函数： `msgsnd/msgrcv`

  + 信号量

    + 头文件： `<sys/sem.h>`
    + IPC对象创建或打开函数： `semget`
	+ IPC控制函数： `semctl`
	+ IPC操作函数： `semop`

  + 共享内存区

    + 头文件： `<sys/shm.h>`
  	+ IPC对象创建或打开函数： `shmget`
	+ IPC控制函数： `shmctl`
	+ IPC操作函数： `shmat/shmdt`

### key_t键和ftok函数

System V IPC使用key_t值作为它们的名字。
头文件<sys/types.h>把key_t这个数据类型定义为一个整数，它通常是一个至少32位的整数，这些整数值通常是由ftok函数赋予的。
函数ftok把一个已存在的路径名和一个整数标识符转换成一个key_t值。

``` c++
#include <sys/ipc.h>

// 成功则返回IPC键，出错则返回-1
key_t ftok(const char *pathname, int id);
```

### ipc_perm结构

内核给每个IPC对象维护一个信息结构，其内容跟内核给文件维护的信息类似：

``` c++
/* Obsolete, used only for backwards compatibility and libc5 compiles */
struct ipc_perm
{
	__kernel_key_t	key;
	__kernel_uid_t	uid;
	__kernel_gid_t	gid;
	__kernel_uid_t	cuid;
	__kernel_gid_t	cgid;
	__kernel_mode_t	mode;
	unsigned short	seq;
};
```
