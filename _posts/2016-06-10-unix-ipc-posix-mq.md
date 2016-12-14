---
layout: post
title: "Posix 消息队列"
description: ""
category: Unix环境编程
tags: [unix ipc]
---
{% include JB/setup %}

消息队列可认为是一个消息链表。有写权限的进程可往队列中放置消息，有读权限的进程可从队列中取走消息。
每个消息都是一个记录，它由发送者赋予一个优先级。
在某个进程往一个队列中写入消息之前，并不需要另外某个进程在该队列上等待消息的到达。
这区别与管道和FIFO，对后两者来说，除非读出者已存在，否则先有写出者是没有意义的。

## 消息队列一种可能的结构模型

![](/images/unix/ipc/posix-mq.png)

## 消息队列的创建、释放

``` c++
#include <mqueue.h>

// 成功则返回消息队列的描述符，失败返回-1
mqd_t mq_open(const char *name, int oflag, ... /* mode_t mode, struct mq_attr *attr */);

// 成功返回0，失败返回-1
int mq_close(mqd_t mqdes);

// 从系统中删除一个消息队列；成功返回0，失败返回-1
int mq_unlink(const char *name);
```

  + mq_open的oflag参数是：O_RDONLY、O_WRONLY、O_RDWR之一，有可能再或上O_CREAT、O_EXCL、O_NONBLOCK；
  + mode参数则指定了消息队列的访问权限，一般为644；
  + attr在创建一个新的消息队列的时候指定消息队列的属性。
    在创建新的消息队列的时候只可指定mq_maxmsg和mq_msgsize属性，属性结构体的另两个成员将被忽略。

同文件系统一样，每个消息队列有一个保存其当前打开着的描述符的引用计数器。
因而mq_unlink能够实现类似于unlink函数删除一个文件的机制：
当一个消息队列的引用计数仍大于0时，其name就能删除，但相应消息队列的析构要等到最后一个mq_close发生时才进行。

## 消息队列属性

``` c++
#include <mqueue.h>

struct mq_attr
{
    long mq_flags; // 消息队列标志
    long mq_maxmsg; // 消息队列中的最大消息数
    long mq_msgsize; // 最大消息长度
    long mq_curmsgs; // 当前消息队列中的消息数
}；

// 成功返回0，失败返回-1
int mq_getattr(int mqdes, struct mq_attr *attr);

// 成功返回0，失败返回-1
int mq_setattr(mqd_t mqdes, const struct mq_attr *newAttr, struct mq_attr *oldAttr);
```

mq_setattr只能设置mq_attr中的mq_flags，以设置或清除非阻塞标志，mq_attr结构的另三个成员将被忽略。

## 消息发送和读取

``` c++
#include <mqueue.h>

// 成功返回0，失败返回-1
int mq_send(int mqdes, const char *buff, size_t len, unsigned int prio);

// 成功则返回消息中的字节数，失败返回-1
ssize_t mq_receive(mqd_t mqdes, char *buff, size_t len, unsigned int *prio);
```

这两个函数分别用于往一个队列中放置一个消息和从一个队列中取走一个消息。
每个消息有一个优先级，它是一个小于MQ_PRIO_MAX的无符号整型数。Posix要求这个值至少为32。

mq_receive总是返回所指定队列中最高优先级的最早消息。
len参数的值不能小于当前队列的mq_msgsize值，否则mq_receive就立即返回EMSGSIZE错误。

## 消息队列非空异步通知机制

在阻塞模式下，当消息队列为空时，调用mq_receive将阻塞调用进程。

Posix消息队列允许异步事件通知，以告知何时有一个消息放置到了某个空消息队列中。这种通知有两种实现方式：

  1. 产生一个信号；
  2. 创建一个线程来执行一个指定的函数

可通过mq_notify为一个进程注册上述形式的通知：

``` c++
#include <mqueue.h>
#include <signal.h>

union sigval
{
    int sival_int;
    void *sival_ptr;
};

struct sigevent
{
    int sigev_notify; // 通知的实现方式(SIGEV_NONE;SIGEV_SIGNAL;SIGEV_THREAD)
    int sigev_signo; // 当sigev_notify为SIGEV_SIGNAL时有效
    union sigval sigev_value; // 传递给信号处理函数或线程函数的参数
    void (*sigev_notify_funciton)(union sigval);  // 线程函数
    pthread_attr_t *sigev_notify_attributes; // 线程属性
};

int mq_notify(mqd_t mqdes, cosnt struct sigevent *notification); // 成功返回0，失败返回-1
```

调用mq_notify为进程注册消息队列非空的通知的一些需注意的规则如下：

  1. notification为空表示撤销当前进程所注册的相应的消息队列非空通知
  2. 任意时刻只有一个进程可以被注册为接收某个给定队列的通知
  3. 当有一个消息到达某个先前为空的队列，而且已有一个进程被注册为接收该队列的通知时，
     只有在没有任何线程阻塞在该队列的mq_receive调用中的前提下，通知才会发出。
     这就是说，在mq_receive调用中的阻塞比任何通知的注册都优先
  4. 当该通知被发送给它的注册进程时，其注册即被撤销。该进程必须再次调用mq_notify以重新注册

## 示例程序

mqdef.h

``` c++
#ifndef __MQDEF_H__
#define __MQDEF_H__

#include <unistd.h>
#include <mqueue.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "common.h"

#define MQNAME "/mqtest"

#endif
```

mqcreate.cpp

``` c++
#include "mqdef.h"

struct mq_attr attr;

int main(int argc, char *argv[])
{
    int maxMsg = 0;
    int msgSize = 0;

    if (argc > 1)
        maxMsg = atoi(argv[1]);
    if (argc > 2)
        msgSize = atoi(argv[2]);

    attr.mq_maxmsg = maxMsg;
    attr.mq_msgsize = msgSize;

    mqd_t mqdes = mq_open(MQNAME, O_RDWR|O_CREAT|O_EXCL, 0644, attr.mq_maxmsg ? &attr : NULL);
    if (mqdes < 0)
        errQuit("create MQ failed.");

    printf("create mq:%s success.\n", MQNAME);
    mq_close(mqdes);
    exit(0);
}
```

mqunlink.cpp

``` c++
#include "mqdef.h"

int main(int argc, char *argv[])
{
    int res = mq_unlink(MQNAME);
    if (res < 0)
        errQuit("unlink mq failed.");
    printf("unlink %s success.\n", MQNAME);
    exit(0);
}
```

mqgetattr.cpp

``` c++
#include "mqdef.h"

int main(int argc, char *argv[])
{
    mqd_t mqdes = mq_open(MQNAME, O_RDONLY);
    if (mqdes < 0)
        errQuit("open mq failed.");

    struct mq_attr attr;
    if (mq_getattr(mqdes, &attr) < 0)
        errQuit("get mq attr failed.");

    PRINT_INTVAL(attr.mq_maxmsg);
    PRINT_INTVAL(attr.mq_msgsize);
    PRINT_INTVAL(attr.mq_curmsgs);

    exit(0);
}
```

mqsend.cpp

``` c++
#include "mqdef.h"

int main(int argc, char *argv[])
{
    mqd_t mqdes = mq_open(MQNAME, O_WRONLY);
    if (mqdes < 0)
        errQuit("open mq failed.");

    struct mq_attr mqattr;
    if (mq_getattr(mqdes, &mqattr) < 0)
        errQuit("get mq attr failed.");

    char *buff = new char[mqattr.mq_msgsize];
    for (int i = 1; i < argc; ++i)
    {
        snprintf(buff, mqattr.mq_msgsize, "%s", argv[i]);
        mq_send(mqdes, buff, strlen(buff), 0);
        printf("mq_send:%s\n", buff);
    }

    mq_close(mqdes);
    exit(0);
}
```

mqrecv.cpp

``` c++
#include "mqdef.h"

int main(int argc, char *argv[])
{
    mqd_t mqdes = mq_open(MQNAME, O_RDONLY);
    if (mqdes < 0)
        errQuit("open mq failed.");

    struct mq_attr attr;
    if (mq_getattr(mqdes, &attr) < 0)
        errQuit("get mq attr faield.");

    int msgCount = 0;
    char *buff = new char [attr.mq_msgsize];
    for (;;)
    {
        unsigned int prio = 0;
        memset(buff, 0, sizeof(char)*attr.mq_msgsize);
        int len = mq_receive(mqdes, buff, attr.mq_msgsize, &prio);
        if (len < 0)
        {
            break;
        }

        printf("recv %d(prio=%d msglen=%d):%s\n", ++msgCount, prio, len, buff);
    }

    mq_close(mqdes);
    exit(0);
}
```

mqrecv1.cpp

``` c++
#include "mqdef.h"

static bool gSigFlag = false;
static void sigUsr1(int signo)
{
    gSigFlag = true;
}

int main(int argc, char *argv[])
{
    mqd_t mqdes = mq_open(MQNAME, O_RDONLY|O_NONBLOCK);
    if (mqdes < 0)
        errQuit("open mq failed.");

    // 设置信号处理程序
    struct sigaction newSa, oldSa;
    newSa.sa_handler = sigUsr1;
    newSa.sa_flags = 0;
    sigemptyset(&newSa.sa_mask);

    if (sigaction(SIGUSR1, &newSa, &oldSa) < 0)
        errQuit("sigaction failed.");

    // 为本进程注册消息队列非空的通知
    struct sigevent ev;
    ev.sigev_notify = SIGEV_SIGNAL;
    ev.sigev_signo = SIGUSR1;

    mq_notify(mqdes, &ev);

    // 接收缓冲区
    struct mq_attr attr;
    mq_getattr(mqdes, &attr);
    char *buff = new char [attr.mq_msgsize];
    unsigned int prio = 0;
    int msgCount = 0;
    memset(buff, 0, sizeof(char)*attr.mq_msgsize);

    sigset_t newMask, oldMask, zeroMask;
    sigemptyset(&newMask);
    sigaddset(&newMask, SIGUSR1);
    sigemptyset(&zeroMask);
    for (;;)
    {
        sigprocmask(SIG_BLOCK, &newMask, &oldMask);
        while (!gSigFlag)
        {
            printf("suspend.\n");
            sigsuspend(&zeroMask);
        }
        printf("comming.\n");

        mq_notify(mqdes, &ev);
        int n = 0;
        while((n=mq_receive(mqdes, buff, attr.mq_msgsize, &prio)) >= 0)
        {
            printf("recv %d (msglen=%d prio=%d): %s\n", ++msgCount, n, prio, buff);
            memset(buff, 0, sizeof(char)*attr.mq_msgsize);
        }

        if (errno != EAGAIN)
        {
            errQuit("recv err.");
        }
        gSigFlag = false;

        printf("end reading.\n");
        sigprocmask(SIG_SETMASK, &oldMask, NULL);
    }

    delete []buff;

    sigaction(SIGUSR1, &oldSa, NULL);
    exit(0);
}
```

mqrecv2.cpp

``` c++
#include "mqdef.h"

static void sigUsr1(int signo) {}

int main(int argc, char *argv[])
{
    mqd_t mqdes = mq_open(MQNAME, O_RDONLY|O_NONBLOCK);
    if (mqdes < 0)
        errQuit("open mq failed.");

    struct sigaction newSa, oldSa;
    newSa.sa_handler = sigUsr1;
    newSa.sa_flags = 0;
    sigemptyset(&newSa.sa_mask);
    sigaction(SIGUSR1, &newSa, &oldSa);

    sigset_t maskSig;
    sigemptyset(&maskSig);
    sigaddset(&maskSig, SIGUSR1);
    sigprocmask(SIG_BLOCK, &maskSig, NULL);

    struct sigevent ev;
    ev.sigev_notify = SIGEV_SIGNAL;
    ev.sigev_signo = SIGUSR1;

    mq_notify(mqdes, &ev);

    struct mq_attr attr;
    int msgCount = 0;
    mq_getattr(mqdes, &attr);
    char *buff = new char [attr.mq_msgsize];
    memset(buff, 0, sizeof(char)*attr.mq_msgsize);

    for (;;)
    {
        int signo;
        sigwait(&maskSig, &signo);
        if (signo == SIGUSR1)
        {
            mq_notify(mqdes, &ev);

            unsigned int prio = 0;
            int n = 0;
            while ((n = mq_receive(mqdes, buff, attr.mq_msgsize, &prio)) >= 0)
            {
                printf("recv %d (msglen=%d  prio=%d): %s\n", ++msgCount, n, prio, buff);
                memset(buff, 0, sizeof(char)*attr.mq_msgsize);
            }

            if (errno != EAGAIN)
                errQuit("recv err.");
        }
    }

    delete []buff;
    buff = NULL;

    exit(0);
}
```

mqrecv3.cpp

``` c++
#include "mqdef.h"

static int gFds[2];
static void sigUsr1(int signo)
{
    write(gFds[1], " ", 1);
}

int main(int argc, char *argv[])
{
    pipe(gFds);

    fd_set rset;
    FD_ZERO(&rset);
    FD_SET(gFds[0], &rset);

    struct sigaction sa;
    sa.sa_handler = sigUsr1;
    sa.sa_flags = 0;
    sigemptyset(&sa.sa_mask);
    if (sigaction(SIGUSR1, &sa, NULL) < 0)
        errQuit("set up signal failed.");

    mqd_t mqdes = mq_open(MQNAME, O_RDONLY|O_NONBLOCK);
    if (mqdes < 0)
        errQuit("open mq failed.");

    struct mq_attr attr;
    mq_getattr(mqdes, &attr);

    int msgCount = 0;
    char *buff = new char [attr.mq_msgsize];
    memset(buff, 0, sizeof(char)*attr.mq_msgsize);

    struct sigevent ev;
    ev.sigev_notify = SIGEV_SIGNAL;
    ev.sigev_signo = SIGUSR1;
    mq_notify(mqdes, &ev);

    for (;;)
    {
        fd_set testSet = rset;
        int nready = select(gFds[0]+1, &testSet, NULL, NULL, NULL);

        if (nready < 0 && errno != EINTR)
            errQuit("select err.");

        if (FD_ISSET(gFds[0], &testSet))
        {
            char c;
            read(gFds[0], &c, 1);
            unsigned int prio = 0;
            int n = 0;
            mq_notify(mqdes, &ev);
            while ((n = mq_receive(mqdes, buff, attr.mq_msgsize, &prio)) >= 0)
            {
                printf("recv %d (msglen=%d prio=%d):%s\n", ++msgCount, n, prio, buff);
                memset(buff, 0, sizeof(char)*attr.mq_msgsize);
            }

            if (n < 0 && errno != EAGAIN)
                errQuit("recv err.");
        }
    }

    exit(0);
}
```

mqrecv4.cpp

``` c++
#include "mqdef.h"

static void threadFunc(sigval_t val);
static int gMsgCount = 0;

int main(int argc, char *argv[])
{
    mqd_t mqdes = mq_open(MQNAME, O_RDONLY|O_NONBLOCK);
    if (mqdes < 0)
        errQuit("open mq failed.");

    struct sigevent ev;
    ev.sigev_notify = SIGEV_THREAD;
    ev.sigev_value.sival_int = mqdes;
    ev.sigev_notify_function = &threadFunc;
    ev.sigev_notify_attributes = NULL;
    if (mq_notify(mqdes, &ev) < 0)
        errQuit("set notify failed.");

    for (;;)
        pause();

    exit(0);
}

static void threadFunc(sigval_t val)
{
    int mqdes = val.sival_int;
    struct mq_attr attr;
    mq_getattr(mqdes, &attr);

    char *buff = new char [attr.mq_msgsize];
    unsigned int prio = 0;
    int n = 0;
    while ((n = mq_receive(mqdes, buff, attr.mq_msgsize, &prio)) >= 0)
    {
        printf("recv %d (msglen=%d prio=%d):%s\n", ++gMsgCount, n, prio, buff);
    }

    if (n < 0 && errno != EAGAIN)
        errQuit("recv err.");
}
```

Makefile

``` shell
SNAIL_ROOT:=../../snail
SNAIL_INC:=$(SNAIL_ROOT)/include
SNAIL_LIB:=$(SNAIL_ROOT)/lib/libsnail.lib

INCDIR:=$(SNAIL_INC)

CC:=clang++
CXX:=clang++
CXXFLAGS:=-g -Wall -I $(subst :, -I,$(INCDIR))
LDFLAGS:=-g -Wall -L $(SNAIL_ROOT)/lib -lsnail -lrt

.PHONY:all clean

all:$(SNAIL_LIB) mqcreate mqunlink mqsend mqgetattr \
     mqrecv mqrecv1 mqrecv2 mqrecv3 mqrecv4

mqcreate:mqcreate.o
     $(CC) $^ $(LDFLAGS) -o mqcreate
mqunlink:mqunlink.o
     $(CC) $^ $(LDFLAGS) -o mqunlink
mqsend:mqsend.o
     $(CC) $^ $(LDFLAGS) -o mqsend
mqrecv:mqrecv.o
     $(CC) $^ $(LDFLAGS) -o mqrecv
mqgetattr:mqgetattr.o
     $(CC) $^ $(LDFLAGS) -o mqgetattr
mqrecv1:mqrecv1.o
     $(CC) $^ $(LDFLAGS) -o mqrecv1
mqrecv2:mqrecv2.o
     $(CC) $^ $(LDFLAGS) -o mqrecv2
mqrecv3:mqrecv3.o
     $(CC) $^ $(LDFLAGS) -o mqrecv3
mqrecv4:mqrecv4.o
     $(CC) $^ $(LDFLAGS) -o mqrecv4

mqcreate.o:mqcreate.cpp mqdef.h
mqunlink.o:mqunlink.cpp mqdef.h
mqsend.o:mqsend.cpp mqdef.h
mqrecv.o:mqrecv.cpp mqdef.h
mqrecv1.o:mqrecv1.cpp mqdef.h
mqrecv2.o:mqrecv2.cpp mqdef.h
mqrecv3.o:mqrecv3.cpp mqdef.h
mqrecv4.o:mqrecv4.cpp mqdef.h
mqgetattr.o:mqgetattr.cpp mqdef.h

$(SNAIL_LIB):
     cd $(SNAIL_ROOT) && $(MAKE)

clean:
     -rm *.o mqcreate mqunlink mqsend mqrecv mqrecv1 mqrecv2 mqrecv3 mqrecv4 mqgetattr
```
