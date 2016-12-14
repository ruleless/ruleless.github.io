---
layout: post
title: "读写锁"
description: ""
category: Unix环境编程
tags: [读写锁, unix同步机制]
---
{% include JB/setup %}

互斥锁把试图进入临界区的所有其他线程都阻塞住。
该临界区通常涉及对由这些线程共享的一个或多个数据的访问或更新。
然而有时候我们可以在读某个数据与修改某个数据之间作区分，以改善并发性能。

**读写锁分配规则**

  1. 当写锁没有被持有时，可以获取任意数量的读锁；
  2. 当读锁、写锁都没有被持有时，可以获取一把写锁。

**创建、销毁**

``` c++
#include <pthread.h>

int pthread_rwlock_init(pthread_rwlock_t *lock, const pthread_rwlockattr_t *attr); // 成功返回0，失败返回-1
int pthread_rwlock_destroy(pthread_rwlock_t *lock); // 成功返回0，失败返回-1
```

也可以通过PTHREAD_RWLOCK_INITIALIZER来初始化读写锁。

**上锁、解锁**

``` c++
#include <pthread.h>

int pthread_rwlock_rdlock(pthread_rwlock_t *lock);
int pthread_rwlock_tryrdlock(pthread_rwlock_t *lock);
int pthread_rwlock_timedrdlock(pthread_rwlock_t *lock, const struct timespec *tm);
int pthread_rwlock_wrlock(pthread_rwlock_t *lock);
int pthread_rwlock_trywrlock(pthread_rwlock_t *lock);
int pthread_rwlock_timedwrlock(pthread_rwlock_t *lock, const struct timespec *tm);
// 以上函数成功返回0，失败返回-1
```

**读写锁属性**

``` c++
#include <pthread.h>

int pthread_rwlockattr_init(pthread_rwlockattr_t *attr);
int pthread_rwlockattr_destroy(pthread_rwlockattr_t *attr);
// 成功返回0，失败返回-1
```

是否用于进程间同步的属性设置和获取函数：

``` c++
#include <pthread.h>

int pthread_rwlockattr_getpshared(pthread_rwlockattr_t *attr, int *bShared);
int pthread_rwlockattr_setpshared(pthread_rwlockattr_t *attr, int bShared);
// 成功返回0，失败返回-1
```

**用互斥锁和条件变量实现读写锁的示例**

rwlock.h

``` c++
#ifndef __RWLOCK_H__
#define __RWLOCK_H__

#include <pthread.h>

class Rwlock {
  public:
    Rwlock() : mutex(PTHREAD_MUTEX_INITIALIZER)
             , wrCond(PTHREAD_COND_INITIALIZER)
             , rdCond(PTHREAD_COND_INITIALIZER)
             , waitWriters(0)
             , waitReaders(0)
             , refCount(0)
    {
    }

    ~Rwlock() {}

    int rdlock();
    int tryrdlock();

    int wrlock();
    int trywrlock();

    int unlock();
  private:
    pthread_mutex_t mutex;
    pthread_cond_t wrCond;
    pthread_cond_t rdCond;

    int waitWriters;
    int waitReaders;
    int refCount;
};

#endif
```

rwlock.cpp

``` c++
#include "rwlock.h"
#include "common.h"
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>

int Rwlock::rdlock()
{
    int res = 0;

    res = pthread_mutex_lock(&mutex);
    if (res != 0) {
        return res;
    }

    ++waitReaders;
    while (0 == res && (-1 == refCount || waitWriters > 0)) // 写入锁被占有，或有等待的写入者
    {
        res = pthread_cond_wait(&rdCond, &mutex);
    }
    --waitReaders;

    if (0 == res) // 获取读出锁
    {
        ++refCount;
    }

    pthread_mutex_unlock(&mutex);
    return res;
}

int Rwlock::tryrdlock()
{
    int res = 0;

    res = pthread_mutex_lock(&mutex);
    if (res != 0)
    {
        return res;
    }

    if (-1 == refCount || waitWriters > 0) // 写入锁被占有，或有等待的写入者
    {
        res = EBUSY;
    }
    else
    {
        ++refCount;
    }

    pthread_mutex_unlock(&mutex);
    return res;
}

int Rwlock::wrlock()
{
    int res = 0;

    res = pthread_mutex_lock(&mutex);
    if (res != 0)
    {
        return res;
    }

    ++waitWriters;
    while (0 == res && refCount != 0) // 等待读写锁空闲
    {
        res = pthread_cond_wait(&wrCond, &mutex);
    }
    --waitWriters;

    if (0 == res) // 获取写入锁
    {
        refCount = -1;
    }

    pthread_mutex_unlock(&mutex);
    return res;
}

int Rwlock::trywrlock()
{
    int res = 0;

    res = pthread_mutex_lock(&mutex);
    if (res != 0)
    {
        return 0;
    }

    if (0 == refCount) // 获取写入锁
    {
        refCount = -1;
    }
    else
    {
        res = EBUSY;
    }

    pthread_mutex_unlock(&mutex);
    return res;
}

int Rwlock::unlock()
{
    int res = 0;

    res = pthread_mutex_lock(&mutex);
    if (res != 0)
    {
        return res;
    }

    // 释放读写锁
    if (-1 == refCount)
    {
        refCount = 0;
    }
    else if (refCount > 0)
    {
        --refCount;
    }
    else
    {
        char szBuff[256] = {0};
        snprintf(szBuff, sizeof(szBuff), "unlock rwlock error! refcount=%d!", refCount);
        errSys(szBuff, false);
    }

    // 通知其他等待着的写入者或读出者
    if (waitWriters > 0)
    {
        if (0 == refCount)
        {
            pthread_cond_signal(&wrCond);
        }
    }
    else if (waitReaders > 0)
    {
        pthread_cond_broadcast(&rdCond);
    }

    pthread_mutex_unlock(&mutex);
    return res;
}
```
