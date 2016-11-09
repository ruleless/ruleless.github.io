---
layout: post
title: "Unix下的基础文件知识"
description: ""
category: Unix环境编程[文件]
tags: [文件]
---
{% include JB/setup %}

## 文件类型

Unix系统的大多数文件是普通文件和目录，但也有另外一些文件类型。文件类型包括如下几种：

  1. **普通文件(regular file)** 这是最常用的文件类型，其类型测试宏为：`S_ISREG`
  2. **目录文件(directory file)**
     对一个目录文件具有读权限的进程可以读该目录的内容，但只有内核可以直接写目录文件。
	 类型测试宏：`S_ISDIR`
  3. **块特殊文件(block special file)** 这种类型的文件提供对设备带缓冲的访问。类型测试宏：`S_ISBLK`
  4. **字符特殊文件(character special file)**
     这种类型的文件提供对设备不带缓冲的访问，每次访问长度可变。
	 系统中的所有设备要么是字符特殊文件，要么是块特殊文件。类型测试宏：`S_ISCHR`
  5. **FIFO(named pipe)有名管道** 通过mkfifo创建，是进程IPC的一种方式。类型测试宏：`S_ISFIFO`
  6. **套接字(socket)** 用于不同主机上的进程通信，也称网络IPC。类型测试宏：`S_ISSOCK`
  7. **符号链接(symbolic link)** 用于指向另一个文件。类型测试宏：`S_ISLINK`

## 文件属性

在Unix平台下，可通过 `struct stat` 访问文件的各项属性，其定义如下：

``` c++
// 文件属性结构
struct stat
{
	mode_t st_mode; // 文件模式字，包括：包括设置用户ID位、设置组ID位、文件访问权限等
	ino_t st_ino; // 文件i节点号
	dev_t st_dev; // device
	numberdev_t st_rdev; // device number for special file
	nlink_t st_nlink; // number of links
	uid_t st_uid; // 文件所属用户ID
	gid_t st_gid; // 文件所属组ID
	off_t st_size; // 文件大小
	time_t st_atime; // time of last access
	time_t st_mtime; // time of last modification
	time_t st_ctime; // time of last file status changeblk
	size_t st_blksize; // best I/O block size
	blckcnt_t st_blocks; // numbers of disk blocks allocated
};
```

属性获取接口声明如下：

``` c++
#include <sys/state.h>

// 文件属性获取函数
int stat(const char *pathname, struct stat *buf); // 成功返回0，失败返回-1
int lstat(const char *pathname, struct stat *buf); // 成功返回0，失败返回-1，此函数可读取符号链接文件的属性
int fstat(const char *pathname, struct stat *buf); // 成功返回0，失败返回-1
```

## 文件模式字(st_mode)

文件模式字包含了文件类型属性、9个文件访问权限位、2个设置ID位和1个粘住位。
文件类型测试宏已在上文定义，此处不再赘述。

  + `S_ISUSR` 设置用户ID位
  + `S_ISGID` 设置组ID位
  + `S_ISVTX` 保存正文（粘住位）
  + `S_IRWXU` 用户读、写、执行权限
  + `S_IRUSR` 用户读权限
  + `S_IWUSR` 用户写权限
  + `S_IXUSR` 用户执行权限
  + `S_IRWXG` 组读、写、执行权限
  + `S_IRGRP` 组读权限
  + `S_IWGRP` 组写权限
  + `S_IXGRP` 组执行权限
  + `S_IRWXO` 其他用户读、写、执行权限
  + `S_IROTH` 其他用户读权限
  + `S_IWOTH` 其他用户写权限
  + `S_IXOTH` 其他用户执行权限

如果程序文件的设置用户ID位被打开，那么使用exec执行该程序时，
新进程的有效用户ID将会是程序文件的属主ID，而非从父进程继承。
一般而言，由普通用户进程 fork 而来的子进程只能访问当前用户目录下的文件。
当普通用户需要创建具有 root 或其他用户权限的进程时，我们就需要设置用户ID位。

文件模式字还包含了文件的访问权限信息。
操作系统通过文件访问权限位和文件属主信息判断一个进程对文件是否有相应的操作权限。
在一个目录下通过"ls -ls"命令会显示如下信息：

``` shell
total 25
  12 -rwxrwxr-x   1 ruleless       ruleless 12642  2015-07-10 08:33 fstat
  3  -rw-rw-r--   1 ruleless       ruleless  2890  2015-07-10 08:33 fstat.cpp
  10 -rw-rw-r--   1 ruleless       ruleless  9924  2015-07-10 08:33 fstat.o
  0  -rw-rw-r--   1 ruleless       ruleless   302  2015-07-09 09:16 makefile
```

第二列显示了文件的访问权限位。

我们可以设置进程的文件模式创建屏蔽字。
文件模式屏蔽字在用户调用 creat 或 open 函数时，将屏蔽在调用参数中指定的文件模式的指定位。
设置文件模式屏蔽字的函数如下：

``` c++
#include <sys/stat.h>

mode_t umask(mode_t cmask); // 设置进程的文件模式创建屏蔽字，返回值：以前的文件模式创建屏蔽字
```

对于已创建的文件，我们可以利用 chmod 更改文件模式。

``` c++
#include <sys/stat.h>

int chmod(const char *pathname, mode_t mode);
int fchmod(int fd, mode_t mode);
	// 以上函数成功返回0，失败返回-1
```

注意：umask 只针对 creat 或 open 创建文件时有效，当我们使用 chmod 函数更改文件访问权限位时，将会忽略 umask 的值。
当调用 umask 时的参数为0时，表示不屏蔽任何访问权限位，我们在创建文件时指定的访问权限位都将被设置。

## 文件属主

文件属主指的是文件所属的用户或组。可利用下述函数更改：

``` c++
#include <sys/stat.h>

int chown(const char *pathname, uid_t uid, gid_t group);
int fchown(int fd, uid_t uid, gid_t group);
int lchown(const char *pathname, uid_t uid, gid_t group);
	// 成功返回0，失败返回-1
```

lchown 与 chown 的区别是，当 lchown 用于符号链接文件时，lchown 更改的是符号链文件的属主，而非符号链接所指向文件的属主。
基于BSD的系统一直规定只有超级用户才能更改一个文件的属主，
这样做的原因是防止用户改变其文件的所有者从而摆脱磁盘空间配额对他们的限制；
系统V则允许任一用户更改他们拥有的文件的属主。

## 文件访问权限测试

文件访问权限测试依下述次序测试：

  1. 若进程有效用户ID为0，则允许访问
  2. 若进程有效用户ID等于文件所属者用户ID，则按用户访问权限位进行访问
  3. 若进程有效组ID等于文件所属者组ID，则按组访问权限位进行访问
  4. 若以上都不匹配，则按其他用户访问权限位进行访问

上面的测试过程依赖的是进程有效用户和文件属主来校验进程对文件是否有指定的访问权限。
另外，Unix 提供了 access 函数，可根据进程实际用户ID来测试进程对文件的访问权限：

``` c++
#include <unistd.h>

int access(const char *pathname, int mode); // 成功返回0，出错返回-1
```

mode参数可取常数值：

  + R_OK 测试读权限
  + W_OK 测试写权限
  + X_OK 测试执行权限
  + F_OK 测试文件是否存在
