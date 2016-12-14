---
layout: post
title: "互斥锁和条件变量"
description: ""
category: Unix环境编程
tags: [互斥锁, 条件变量, unix同步机制]
---
{% include JB/setup %}

互斥锁用于保护临界区，以保证任何时刻只有一个线程在执行其中的代码。
条件变量用于等待互斥锁、并在特定时刻唤醒它。能用互斥锁和条件变量解决的进程或线程的同步和交互问题一般也能用信号量解决。
互斥锁是为上锁而优化的，条件变量是为等待而优化的，信号量既可用于上锁，也可用于等待，因而可能导致更多的开销和更高的复杂性。

## 互斥锁

### 创建、销毁

``` c++
#include <pthread.h>

int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *attr); // 成功返回0，失败返回错误码
int pthread_mutex_destroy(pthread_mutex_t *mutex); // 成功返回0，失败返回错误码
```

分配在数据段或栈区的互斥锁可以用PTHREAD_MUTEX_INITIALIZER初始化，也可以用pthread_mutex_init初始化；
而分配在堆区的互斥锁则只能用pthread_mutex_init初始化。

用pthread_mutex_init初始化的互斥锁最好用pthread_mutex_destroy销毁，而不管它是否分配于堆区。

pthread_mutex_init的第二个参数是互斥锁属性，目前它仅用于指示互斥锁是用于线程同步还是进程同步。

### 上锁、解锁

``` c++
#include <pthread.h>

int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
// 上述函数成功返回0，出错返回错误码
```

pthread_mutex_lock给mutex上锁，若该mutex已被锁住，则阻塞，直到mutex被解锁。

pthread_mutex_trylock是pthread_mutex_lock的非阻塞版本，若mutex已被锁住，该函数返回EBUSY。

pthread_mutex_unlock给mutex解锁。


### 互斥锁属性

``` c++
#include <pthread.h>

int pthread_mutexattr_init(pthread_mutexattr_t *attr); // 成功返回0，失败返回错误码
int pthread_mutexattr_destroy(pthread_mutexattr_t *attr); // 成功返回0，失败返回错误码
```

进程间共享属性：

``` c++
enum
{
    PTHREAD_PROCESS_PRIVATE, // 线程间共享
#define PTHREAD_PROCESS_PRIVATE PTHREAD_PROCESS_PRIVATE
    PTHREAD_PROCESS_SHARED // 进程间共享
#define PTHREAD_PROCESS_SHARED  PTHREAD_PROCESS_SHARED
};

int pthread_mutexattr_getpshared(const pthread_mutexattr_t *attr, int *val);
int pthread_mutexattr_setpshared(pthread_mutexattr_t *attr, int val);
// 成功返回0，失败返回-1
```

## 条件变量

### 创建、销毁

``` c++
#include <pthread.h>

int pthread_cond_init(pthread_cond_t *cond, const pthread_condattr_t *attr); // 成功返回0，失败返回-1
int pthread_cond_destroy(pthread_cond_t *cond); // 成功返回0，失败返回-1
```

条件变量pthread_cond_t在Linux实现下的定义如下：

``` c++
typedef union
{
    struct
    {
        int __lock;
        unsigned int __futex;
        __extension__ unsigned long long int __total_seq;
        __extension__ unsigned long long int __wakeup_seq;
        __extension__ unsigned long long int __woken_seq;
        void *__mutex;unsigned int __nwaiters;
        unsigned int __broadcast_seq;
    } __data;
    char __size[__SIZEOF_PTHREAD_COND_T];
    __extension__ long long int __align;
} pthread_cond_t;
```

pthread_cond_t 可通过PTHREAD_COND_INITIALIZER初始化，也可通过pthread_cond_init初始化。

### 等待和唤醒

``` c++
#include <pthread.h>

// 等待函数，成功返回0，失败返回-1
int pthread_cond_wait(pthread_mutex_t *mutex, pthread_cond_t *cond);
int pthread_cond_timedwait(pthread_mutex *mutex, pthread_cond_t *cond, const struct timespec *abstime);

// 唤醒函数，成功返回0，失败返回-1
int pthread_cond_signal(pthread_cond_t *cond);
int pthread_cond_broadcast(pthread_cond_t *cond);
```

### 条件变量属性

``` c++
#include <pthread.h>

int pthread_condattr_init(pthread_condattr_t *condattr); // 成功返回0，失败返回-1
int pthread_condattr_destroy(pthread_condattr_t *condattr); // 成功返回0，失败返回-1
```

进程间共享属性

``` c++
int pthread_condattr_getpshared(pthread_condattr_t *condattr, int *shared); // 成功返回0，失败返回-1
int pthread_condattr_setpshared(pthread_condattr_t *condattr, int shared); // 成功返回0，失败返回-1
```

## 生产者-消费者问题

SampleDef.h

``` c++
#ifndef __SAMPLEDEF_H__
#define __SAMPLEDEF_H__

#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define N 1000000
#define MAX_THREAD 10

struct Shared
{
    int buff[N];
    int nput;
    int nputval;
    pthread_mutex_t producerMutex;

    int nReady;
    pthread_mutex_t readyMutex;
    pthread_cond_t cond;

    Shared()
    {
        memset(buff, -1, sizeof(buff));
        nput = 0;
        nputval = 0;
        nReady = 0;
    }
};

#endif
```

``` c++
// 多生产者-单消费者
#include <pthread.h>
#include "common.h"

#include "SampleDef.h"

int gItems = 0;
Shared gShared;

void* producer(void *arg);
void* consumer(void *arg);

int main(int argc, char *argv[])
{
    gItems = 1;
    if (argc > 1)
        gItems = atoi(argv[1]);

    int nProducer = MAX_THREAD;
    if (argc > 2)
        nProducer = min(nProducer, atoi(argv[2]));

    pthread_t tidProducers[MAX_THREAD];
    int execTimes[MAX_THREAD];
    for (int i = 0; i < nProducer; ++i)
    {
        execTimes[i] = 0;
        pthread_create(&tidProducers[i], NULL, producer, &execTimes[i]);
    }

    pthread_t tidConsumer;
    pthread_create(&tidConsumer, NULL, consumer, NULL);

    for (int i = 0; i < nProducer; ++i)
    {
        pthread_join(tidProducers[i], NULL);
    }
    pthread_join(tidConsumer, NULL);

    for (int i = 0; i < nProducer; ++i)
    {
        printf("producer %d run %d times\n", i, execTimes[i]);
    }

    exit(0);
}

void* producer(void *arg)
{
    for (;;)
    {
        pthread_mutex_lock(&gShared.producerMutex);
        if (gShared.nputval >= gItems)
        {
            pthread_mutex_unlock(&gShared.producerMutex);
            return NULL;
        }
        gShared.buff[gShared.nput++%N] = gShared.nputval++;
        pthread_mutex_unlock(&gShared.producerMutex);

        pthread_mutex_lock(&gShared.readyMutex);
        ++gShared.nReady;
        pthread_mutex_unlock(&gShared.readyMutex);

        if (1 == gShared.nReady)  // 唤醒消费线程
            pthread_cond_signal(&gShared.cond);

        (*((int *)arg))++;
    }
    return NULL;
}

void* consumer(void *arg)
{
    for (int i = 0; i < gItems; ++i)
    {
        pthread_mutex_lock(&gShared.readyMutex);
        while (gShared.nReady <= 0)
        {
            pthread_cond_wait(&gShared.cond, &gShared.readyMutex);
        }

        --gShared.nReady;
        if (gShared.buff[i%N] != i)
        {
            printf("conflict! index=%d curval=%d legalval=%d\n", i%N, gShared.buff[i%N], i);
        }

        pthread_mutex_unlock(&gShared.readyMutex);
    }
    return NULL;
}
```

``` c++
// 多生产者-多消费者
#include <pthread.h>
#include "SampleDef.h"
#include "common.h"

int gItems = 1;
Shared gShared;

void* producer(void *arg);
void* consumer(void *arg);

int main(int argc, char *argv[])
{
    if (argc > 1)
        gItems = min(atoi(argv[1]), N);

    int nProducer = MAX_THREAD;
    if (argc > 2)
        nProducer = min(atoi(argv[2]), MAX_THREAD);
    int nConsumer = MAX_THREAD;
    if (argc > 3)
        nProducer = min(atoi(argv[3]), MAX_THREAD);

    pthread_t tidProducers[MAX_THREAD];
    int execTimesOfProducer[MAX_THREAD];
    for (int i = 0; i < nProducer; ++i)
    {
        execTimesOfProducer[i] = 0;
        pthread_create(&tidProducers[i], NULL, producer, &execTimesOfProducer[i]);
    }

    pthread_t tidConsumers[MAX_THREAD];
    int execTimesOfConsumer[MAX_THREAD];
    for (int i = 0; i < nConsumer; ++i)
    {
        execTimesOfConsumer[i] = 0;
        pthread_create(&tidConsumers[i], NULL, consumer, &execTimesOfConsumer[i]);
    }

    for (int i = 0; i < nProducer; ++i)
    {
        pthread_join(tidProducers[i], NULL);
    }

    for (int i = 0; i < nConsumer; ++i)
    {
        pthread_join(tidConsumers[i], NULL);
    }

    for (int i = 0; i < nProducer; ++i)
    {
        printf("producer %d run %d times\n", i, execTimesOfProducer[i]);
    }
    for (int i = 0; i < nConsumer; ++i)
    {
        printf("consumer %d run %d times\n", i, execTimesOfConsumer[i]);
    }

    exit(0);
}

void* producer(void *arg)
{
    for (;;)
    {
        pthread_mutex_lock(&gShared.producerMutex);
        if (gShared.nputval >= gItems)
        {
            pthread_mutex_unlock(&gShared.producerMutex);
            return NULL;
        }
        gShared.buff[gShared.nput++] = gShared.nputval++;
        pthread_mutex_unlock(&gShared.producerMutex);

        pthread_mutex_lock(&gShared.readyMutex);
        ++gShared.nReady;
        pthread_mutex_unlock(&gShared.readyMutex);

        if (1 == gShared.nReady)
            pthread_cond_broadcast(&gShared.cond);

        (*((int *)arg))++;
    }
    return NULL;
}

void* consumer(void *arg)
{
    for (;;)
    {
        pthread_mutex_lock(&gShared.readyMutex);
        while (gShared.nReady <= 0 && gShared.ngetval < gItems)
        {
            pthread_cond_wait(&gShared.cond, &gShared.readyMutex);
        }
        if (gShared.ngetval >= gItems)
        {
            pthread_mutex_unlock(&gShared.readyMutex);
            return NULL;
        }

        --gShared.nReady;
        if (gShared.buff[gShared.nget] != gShared.ngetval)
        {
            printf("conflict! index=%d curval=%d legalval=%d\n", gShared.nget, gShared.buff[gShared.nget], gShared.ngetval);
        }
        ++gShared.nget;
        ++gShared.ngetval;

        pthread_mutex_unlock(&gShared.readyMutex);

        (*((int *)arg))++;
    }
    return NULL;
}
```
