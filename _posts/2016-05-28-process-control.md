---
layout: post
title: "Unix环境下的进程控制"
description: ""
category: Unix环境编程
tags: [进程]
---
{% include JB/setup %}

## 进程控制接口

### 创建子进程

``` c++
#include <unistd.h>

// 子进程中返回0，在父进程中返回子进程ID，出错返回-1
int fork();

// vfork与fork在语义上一致，但vfork不会复制父进程空间，
// 用vfork创建的子进程以与父进程共享地址空间的方式运行；
// 另外调用vfork后父进程将会阻塞，直到子进程调用exit或exec为止
int vfork();
```

### 执行新程序

``` c++
#include <unistd.h>

int execl(cosnt char* pathname, const char* arg0, ..., (const char*)0);
int execle(const char* pathname, const char* arg0, ..., (const char*)0, const char* env);
int execlp(const char* filename, const char* arg0, ..., (const char*)0);
int execv(const char* pathname, const char *argv[]);
int execve(const char* pathname, const char* argv[], const char* env);
int execvp(const char* filename, const char* argv[]);

// 上述函数以一个新的程序替换当前进程空间（包括代码段、数据段、堆、栈段）。若失败返回-1，成功不返回。
```

所有exec函数中只有execve是系统调用，其他都是库函数。六个exec函数的关系如下：

![](/images/unix/process/process-exec.png)

### 进程同步

``` c++
#include <sys/wait.h>

int wait(int *status); // 阻塞，直到任意进程返回

// 阻塞，直到指定进程返回；另可设置options使其不阻塞
// 返回值：若成功返回进程ID，若失败返回-1
int waitpid(pid_t pid, int *status, int options);
```

waitpid 的 pid 参数说明如下：

  + `pid==-1` 等待任意子进程
  + `pid>0` 等待进程ID等于pid的子进程
  + `pid==0` 等待进程组ID等于调用进程组ID的任意子进程
  + `pid<0` 等待进程组ID等于pid绝对值的任意子进程

### 进程ID获取接口

``` c++
#include <unistd.h>

pid_t getpid(); // 返回调用进程的进程ID
pid_t getppid(); // 返回调用进程的父进程ID
```

### 基本进程控制原语

fork,exec,exit,wait组成了基本的进程控制原语：fork创建新进程，exec执行新程序，exit处理终止，wait等待终止。

### 示例

下面给出一个创建子进程，并调用 exec 执行一段新程序的示例。
示例本身没有任何意义，纯粹为了掩饰上述函数的调用。

cat.cpp 文件：

``` c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>

static const int s_buffSize = 8;
char buff[s_buffSize];

int main(int argc, char* argv[])
{
	int n = 0;
	while ((n=read(STDIN_FILENO, buff, s_buffSize)) > 0)
	{
		if (write(STDOUT_FILENO, buff, n) != n)
		{
			printf("write error! pid=%d\n", getpid());
			exit(1);
		}
	}
	exit(0);
}
```

fork.cpp 文件：

``` c++
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

static const int s_maxChildP = 10;

int main(int argc, char *argvp[])
{
	pid_t childPid = -1;
	if ((childPid = fork()) < 0)
	{
		printf("fork error!\n");
		abort();
	}
	else if (childPid == 0) // 子进程
	{
		execlp("./cat", "cat", NULL);
	}

	waitpid(childPid, NULL, 0);

	exit(0);
}
```

## 僵尸进程

若子进程结束时，父进程未对其进行善后处理(waitpid)，那么该子进程就成为了僵尸进程。

若子进程的父进程先于自己结束，那么该子进程讲会被init(pid=1)进程领养。

若父进程不想等待子进程，而子进程又不至沦为僵尸进程的处理方式是：
调用fork函数两次，一次在父进程中调用，一次在子进程中调用。
父进程在 fork 之后等待子进程，而子进程在 fork 之后迅速结束，孙子进程则被 init 领养。
这样，就不会产生僵尸进程。下面是具体的代码示例：

``` c++
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(int argc, char *argv[])
{
	int pid = -1;
	if ((pid = fork()) < 0)
	{
		printf("fork error.\n");
		abort();
	}
	else if(pid == 0)
	{
		if ((pid = fork()) < 0)
		{
			printf("fork error.\n");
		}
		else if(pid > 0)
			exit(0);

		sleep(2);
		printf("child process. pid=%d  ppid=%d\n", getpid(), getppid());
		exit(0);
	}

	printf("parent process. pid=%d\n", getpid());

	exit(0);
}
```
