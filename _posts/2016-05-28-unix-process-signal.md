---
layout: post
title: "Unix信号"
description: ""
category: Unix环境编程
tags: [unix进程]
---
{% include JB/setup %}

信号是Posix标准的重要内容，它提供一种处理异步事件的方法。

## 不可靠信号

信号不可靠指的是：信号可能丢失，信号可控制性差。

信号处理程序设置函数为：

``` c++
#include <signal.h>

void (*signal(int signo, void (*handler)(int)))(int);
// 此声明较为晦涩，其等价于：
typedef void (*SigHandler)(int);
SigHandler signal(int signo, SigHandler handler);
// 成功返回该信号之前的处理程序，出错返回SIG_ERR
```

对于任何一个信号都只能按下列三种方式中的一种来执行：

  1. 忽略(`SIG_IGN`)
  2. 执行系统默认动作(`SIG_DFL`)
  3. 调用配置的信号处理函数

在早期版本中，当通过配置的信号处理函数处理产生的信号时，该信号动作将被复位为默认值。
我们不得不在信号处理程序中重新配置信号处理程序。如下：

``` c++
...
signal(SIG_INT, sig_int);
...
void sig_int()
{
	signal(SIG_INT, sig_int);
	...;
}
```

这种代码有个问题，在发生中断信号之后与在sig_int中调用signal之前有一个时间窗口，
若在此时间窗口中再次发生中断信号。那么第二个中断信号将会导致执行默认动作，
而SIG_INT的默认动作是终止进程。

## 发送自定义信号

信号不仅产生自内核，我们在用户态应用进程内也可自发向自身或其他进程发送信号。

信号发送函数：

``` c++
#include <signal.h>

int kill(pid_t pid, int signo); // 成功返回0，失败返回-1
int raise(int signo); // 成功返回0，失败返回-1(向自身发送信号)
```

kill 函数的 pid 参数取值如下：

  + `pid>0` 将信号发送给进程ID等于pid的进程
  + `pid==0` 将信号发送给调用进程所在组的所有进程
  + `pid==-1` 将信号发送给调用进程所有有权限发送信号的进程
  + `pid<-1` 将信号发送给进程组ID等于pid绝对值的进程组

进程将信号发送给其他进程需要权限，其规则是：

  1. root用户进程(实际用户ID等于0)可将信号发送给任何进程；
  2. 发送者的实际用户ID或有效用户ID需等于接收者的实际用户ID或有效用户ID。

### alarm函数

alarm函数在指定时间之后为进程产生一个SIGALRM信号。若指定seconds为0，则不会产生SIGALRM信号。

``` c++
#include <unistd.h>

unsigned int alarm(unsigned int seconds); // 0或以前设置的闹钟时钟余留值
```

### pause函数

调用pause函数将使进程挂起，直到捕捉到一个信号。该函数总是返回-1，并将errno设置为EINTR。

``` c++
#include <unistd.h>

int pause();
```

## 可靠信号机制

可靠信号机制引入了未决信号、递送等可靠信号术语。

一种可能的信号递送实现：

![](/images/unix/process/available-signal.png)

在可靠信号机制中，所有产生的信号都先进入未决状态，进程可以选择处理处于未决状态的信号(递送)或将其阻塞。

### 信号集

信号集类型定义为sigset_t，其操作函数如下：

``` c++
#include <signal.h>

int sigemptyset(sigset_t *set); // 清空信号集
int sigfillset(sigset_t *set); // 填充信号集
int sigaddset(sigset_t *set, int signo);
int sigdelset(sigset_t *set, int signo);
// 以上四个函数若成功则返回0，失败返回-1

int sigismember(const sigset* set, int signo); // 若signo是set成员则返回1，否则返回0，出错返回-1
```

### 信号设置函数

``` c++
#include <signal.h>

struct sigaction {
	void (*sa_handler)(int signo); // 信号处理函数
	int sa_mask; // 进入信号处理函数时的信号屏蔽字，返回时会恢复

	int sa_flags;
	void (*sa_sigaction)(int signo, siginfo_t *info, void *context);
};

int sigaction(int signo, const struct sigaction *newact, struct sigaction *oldact); // 成功返回0，失败返回-1
```

相比signal此函数除设置信号处理程序之外还拥有的特性包括：

  1. 在内核向进程递送信号之后，信号执行动作不会被复位
  2. 执行信号处理程序时，指定的sa_mask与当前信号都会被屏蔽，这样就可以保证信号处理程序不被我们所不期望的信号所中断
  3. 可指定被信号中断的系统调用在信号递送之后的行为(中断或重启)

sa_flags常用常值定义如下：

  + `SA_INTERRUPT` 由此信号中断的系统调用不会自动重启动
  + `SA_RESTART` 由此信号中断的系统调用将会自动重启动

### 进程信号屏蔽字操作接口

``` c++
#include <signal.h>

// 设置、获取进程的信号屏蔽字
int sigprocmask(int how, const sigset_t *newmeask, sigst *oldmask); // 成功返回0，失败返回-1

// 获取进程当前的信号屏蔽字
int sigpending(sigset_t *mask); // 成功返回0，失败返回-1
```

how可取值：

  + `SIG_BLOCK` 新的信号屏蔽字为原先屏蔽字与newmask的并集
  + `SIG_UNBLOCK` 新的信号屏蔽字为原先屏蔽字与newmask的差集
  + `SIG_SETMASK` 新的信号屏蔽字为newmask

sigprocmask常用来保护不希望由信号中断的代码临界区。

### 进程挂起

与 pause 类似，新的可靠信号利用 sigsuspend 函数挂起进程。

``` c++
#include <signal.h>

// 将进程信号屏蔽字设为mask，然后挂起直到捕捉到一个信号，返回值：-1，并将errno设为EINTR
int sigsuspend(cosnt sigset_t *mask);
```
