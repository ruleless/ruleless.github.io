---
layout: post
title: "Unix环境下的基础进程知识"
description: ""
category: Unix环境编程
tags: [进程]
---
{% include JB/setup %}

## C程序内存布局

![](/images/unix/process/c-memstructure.png)

## 进程退出函数

``` c++
#include <stdlib.h>

void _exit(int status);
void _Exit(int status);
voit exit(int status);
void atexit(void (*func)());  // 注册进程退出时调用的函数
```

## 指令跳转机制

使用goto语句即可实现函数内跳转，此种跳转只需修改指令指针，不需修改堆栈指针。
而如果需要在函数之间跳转则复杂得多。

Unix下实现函数之间跳转的接口定义如下：

``` c++
#include <setjmp.h>

int setjmp(jmp_buf env);  // 第一次返回0，调用longjmp跳转至此处时返回longjmp所设置的参数
void longjmp(jmp_buf env, int res); // 跳转至setjmp处
```

进行非函数内跳转时，需修改指令指针和堆栈指针，
指令指针用于重新定位程序执行位置，堆栈指针用于恢复执行环境。
有一个问题？在longjmp之后，函数栈是否会完全恢复到第一次调用setjmp时的样子
（亦即，栈内的局部变量是否会回滚）。
Posix标准并未对此作答，但我们却可以用volatile关键字来保证局部变量不回滚；
那如果我们就想回滚怎么办？没办法！

一个简单的跳转示意程序如下：

``` c++
#include <unistd.h>
#include <setjmp.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static jmp_buf jmpbuffer;

void jumpByCondition(char* str);

int main(int argc, char *argv[])
{
    const int s_maxBuffer = 256;
    char buff[s_maxBuffer];
    int val = 0;
    register int regVal = 0;
    volatile int volVal = 0;

    if (setjmp(jmpbuffer) != 0)
    {
        printf("error val=%d, regVal=%d, volVal=%d\n", val, regVal, volVal);
        exit(1);
    }

    while (fgets(buff, s_maxBuffer, stdin))
    {
        ++val; ++regVal; ++volVal;
        jumpByCondition(buff);
    }

    exit(0);
}

void jumpByCondition(char* str)
{
    if (strstr(str, "jump") != NULL )
    {
        longjmp(jmpbuffer, 1);
    }
    else
    {
        int len = strlen(str);
        if (len - 1 > 0 && str[len - 1] == '\n')
            str[len - 1] = '\0';
        puts(str);
    }
}
```

## 进程资源

我们可通过如下接口获得和设置一个进程所拥有的资源：

``` c++
#include <sys/resource.h>

int getrlimit(int resource, struct rlimit *pLimit); // 成功返回0，出错返回非0
int setrlimit(int resource, const struct rlimt *pLimit); // 成功返回0，出错返回非0

struct rmlit
{
	int rlim_cur; // 软限制（当前限制）
	int rlim_max; // 硬限制（最大限制）
};
```

参数 resource 可取如下常量：

  + `RLIMIT_AS` 进程可用存储区的最大长度
  + `RLIMIT_CORE` core文件的最大字节数，若为0则表示阻止创建core文件
  + `RLIMIT_CPU` CPU时间的最大量值（单位秒），若超过此限制则向进程发送SIGXCPU信号
  + `RLIMIT_DATA` 初始化数据段、BSS段、堆的总长度限制
  + `RLIMIT_FSIZE` 可创建的文件的最大长度，超过此长度时向进程发送SIGXFSZ信号
  + `RLIMIT_LOCKS` 一个进程可持有文件锁的最大数量
  + `RLIMIT_MEMLOCK` 进程使用mlock可锁定的最大字节长度
  + `RLIMIT_NOFILE` 进程能打开的最大文件数
  + `RLIMIT_NPPROC` 每个用户ID可拥有的最大子进程数
  + `RLIMIT_RSS` 最大驻内存集的字节长度
  + `RLIMIT_SBSIZE` 用户在任一给定时间内可以占用的套接字缓冲区的最大长度
  + `RLIMIT_STACK` 栈的字节长度
  + `LIMIT_VMEM` RLIMIT_AS的同意词

我们可用如下程序打印进程的资源限制：

``` c++
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/resource.h>

#define PRINT_RLIMIT(res) printrlimit(#res, res)
void printrlimit(const char* name, int resID);

int main(int argc, char *argv[])
{
	PRINT_RLIMIT(RLIMIT_AS);
	PRINT_RLIMIT(RLIMIT_CORE);
	PRINT_RLIMIT(RLIMIT_CPU);
	PRINT_RLIMIT(RLIMIT_DATA);
	PRINT_RLIMIT(RLIMIT_FSIZE);
	PRINT_RLIMIT(RLIMIT_LOCKS);
	PRINT_RLIMIT(RLIMIT_MEMLOCK);
	PRINT_RLIMIT(RLIMIT_NOFILE);
     #ifdef RLIMIT_NPPROC
	PRINT_RLIMIT(RLIMIT_NPPROC);
     #endif
	PRINT_RLIMIT(RLIMIT_RSS);
     #ifdef RLIMIT_SBSIZE
	PRINT_RLIMIT(RLIMIT_SBSIZE);
     #endif
	PRINT_RLIMIT(RLIMIT_STACK);
     #ifdef RLIMIT_VMEM
	PRINT_RLIMIT(RLIMIT_VMEM);
     #endif
	exit(0);
}

void printrlimit(const char* name, int resID)
{
	struct rlimit limit;
	if (getrlimit(resID, &limit) != 0)
	{
		printf("getrlimit error. resid=%d\n", resID);
		exit(1);
	}

	printf("%14s    curlim:", name);
	if (limit.rlim_cur == RLIM_INFINITY)
	{
		printf("(infinity)");
	}
	else
	{
		printf("%10ld", limit.rlim_cur);
	}

	printf("    maxlim:");
	if (limit.rlim_max == RLIM_INFINITY)
	{
		printf("(infinity)");
	}
	else
	{
		printf("%10ld", limit.rlim_max);
	}
	putchar('\n');
}
```
