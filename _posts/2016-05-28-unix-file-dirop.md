---
layout: post
title: "Unix下的目录操作"
description: ""
category: Unix环境编程
tags: [unix文件]
---
{% include JB/setup %}

## 读目录接口

``` c++
#include <dirent.h>

struct dirent
{
	ino_t d_ino; // i节点号
	char d_name[256]; // 文件名
};

DIR *opendir(const char *pathname); // 打开目录，失败返回空指针
struct dirent *readdir(DIR *dir); // 读目录，此函数将自动迭代，依次返回该目录下的所有文件
int closedir(DIR *dir); // 关闭目录

long telldir(DIR *dir); // 返回与dir关联的目录中的当前位置
void seekdir(DIR *dir, long loc);
```

## 创建和删除目录

创建目录接口：

``` c++
#include <sys/stat.h>

int mkdir(const char *pathname, mode_t mode); // 创建目录。成功返回0，失败返回-1
```

此函数创建一个新的空目录，其中，.和..目录项是自动创建的。创建出来的访问权限为：mode&(~umask)。
一般普通文件的访问权限为：rw-rw-r--(664)，一般普通目录的访问权限为：rwxrwxr-x(775)。
相比文件，一般而言目录需要多增加一个执行位。这是因为，我们用名字打开任一类型的文件时，
对该名字中包含的每一个目录都应具有执行权限，所以执行位也通常称为搜索位。

删除目录的接口：

``` c++
#include <unistd.h>

int rmdir(const char *pathname); // 删除目录。成功返回0，失败返回-1
```

如果调用此函数使目录的链接计数成为0，并且也没有其他进程打开此目录，则释放由此目录占用的空间。
如果在链接计数达到0时，有一个或几个进程打开了此目录，则在此返回前删除最后一个链接及.和..项。
另外，在此目录中不能再创建新文件。但是在最后一个进程关闭它之前并不释放此目录。
（即使另一个进程打开该目录，它们在此目录下也不能执行其他操作。
这样处理的原因是，为了使rmdir函数成功执行，该目录必须是空的。）

## 切换目录

``` c++
#include <unistd.h>

int chdir(const char *pathname); // 成功返回0，失败返回-1
int fchdir(int fd); // 成功返回0，失败返回-1
char* getcwd(char *buf, size_t size); // 获取当前工作目录的绝对路径，成功返回buf，失败返回NULL
```

## Unix平台下遍历目录的示例程序

``` c++
typedef bool (*FileHandler)(const char *pathname);
int traverseDir(const char *pathname, FileHandler handler)
{
	struct stat fileStat;
	if (lstat(pathname, &fileStat) < 0)
	{
		return -1;
	}

	if (!handler(pathname))
	{
		return 0;
	}
	if (!S_ISDIR(fileStat.st_mode))
	{
		return 0;
	}

	DIR *dir = opendir(pathname);
	if (dir != NULL)
	{
		char subpath[NAME_PATH+1];
		memset(subpath, 0, sizeof(subpath));
		strncpy(subpath, pathname, NAME_PATH);
		char *ptr = subpath;
		ptr += strlen(subpath);
		if (ptr > subpath && *(ptr-1) != '/')
		{
			*ptr++ = '/';
		}

		struct dirent *ent = readdir(dir);
		while (ent != NULL)
		{
			if (strcmp(ent->d_name, ".") == 0 || strcmp(ent->d_name, "..") == 0)
			{
				ent = readdir(dir);
				continue;
			}

			strncpy(ptr, ent->d_name, NAME_PATH-(int)ptr+(int)subpath);
			*(ptr+strlen(ent->d_name)) = '\0';
			traverseDir(subpath, handler);

			ent = readdir(dir);
		}
		closedir(dir);
	}

	return 0;
}
```
