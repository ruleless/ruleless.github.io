---
layout: post
title: "常用Linux命令和工具"
description: ""
category: linux
tags: [linux]
---
{% include JB/setup %}

## 普通用户常用命令或工具

### 常用命令

  + `cd` 更改当前目录
  + `ls` 显示目录下的文件
  + `find` 根据文件名进行查找
  + `grep` 查找文件内容
  + `chown` 更改文件属主
  + `chmod` 更改文件模式位
  + `ps` 显示进程快照
  + `ln` 创建文件链接
    - `ln target hardlink` 创建硬链接
	- `ln -s target symbollink` 创建符号链接
  + `history` 显示历史命令
  + `tar` 打包工具
    - `tar -cvf target.tar file1 file2` 将file1和file2归档到target.tar
	- `tar -xvf target.tar` 文件解包
  + `unzip` 解压缩zip文件
  + `zip` 压缩zip文件
  + `alias` 创建命令别名
  + `su` 切换用户
  + `sudo` 临时使用管理员权限
  + `file` 查看文件类型

### 磁盘管理

  + `du` 查看文件所占用的磁盘空间
  + `df` 查看磁盘空间使用情况
  + `fdisk` 磁盘分区表管理工具，可用于查看和划分磁盘分区
  + `mkfs` 创建Linux文件系统
  + `mount` 文件系统挂载工具

### 用户管理命令

  + `useradd` 添加用户
  + `passwd` 修改用户命令
  + `who` 查看当前登录到该系统的用户
  + `finger` 查看当前登录到该系统的用户

### 系统信息查看命令

  + `top` 实时显示当前进程运行状态
  + `free` 显示当前系统内存使用情况
  + `/proc/version /proc/cpuinfo /proc/meminfo` 在/proc目录下会有很多跟系统信息相关的临时文件

### 网络管理命令

  + `ip` 新的网络配置管理和查看命令，旧的 `ifconfig` 不再使用。例：`ip addr` 查看网卡地址
  + `ss` 查看当前网络活动，旧的 `netstat` 不再使用。例：`ss -anl` 查看系统当前TCP监听端口
  + `route` 路由表管理命令。
    - `route -n` 显示路由表
	- `route del -net 0.0.0.0` 删除默认路由
	- `route add -net 0.0.0.0 gw 10.0.2.2 dev eth0` 添加默认路由
  + `arp` ARP高速缓存管理
    - `arp -d 192.168.56.100` 删除高速缓存
  + `tcpdump` 抓包工具
  + `traceroute` 网络包追踪

### selinux(Security-Enhanced Linux)

跟 selinux 相关的命令有 getenforce 和 setenforce。具体可查看手册页：man selinux

### openssl加解密工具

**RSA加解密**

  + 生成RSA私钥：`openssl genrsa -des3 -out private.pem 2048`
  + 生成RSA私钥对应的公钥：`openssl rsa -in private.pem -outform PEM -pubout -out public.pem`
  + 使用RSA公钥加密文件：`openssl rsautl -encrypt -inkey public.pem -pubin -in file.in -out file.rsa`
  + 使用RSA私钥解密文件：`openssl rsautl -decrypt -inkey private.pem -in file.rsa -out file.out`

## 开发工具或命令

### 编译相关

  + `gcc/g++` GNU C/C++编译器
    - `gcc -E source.c -o target.i` 预处理：展开宏
	- `gcc -S target.i -o target.s` 编译：将源文件编译为汇编代码
	- `gcc -c target.s -o target.o` 汇编：将汇编代码编译为机器码(中间目标文件)
	- `gcc target.o -o target` 链接：将目标文件链接为可执行文件
  + `ar` 目标文件打包工具，可用于打包静态库
  + `make` 软件工程自动化构建工具
  + `gdb` 软件调试工具
  + `ld` GND链接器
  + `ldd` 显示程序所需的动态链接库
  + `nm` 打印目标文件的符号信息
  + `strings` 查看嵌入于二进制文件中的字符串
  + `time` 显示程序执行时间

### 性能分析及程序追踪工具

  + `strace` 追踪软件使用的系统调用
  + `valgrind` 应用程序调试及分析工具包
    - `valgrind --leak-check=full a.out` 内存泄漏检查
