---
layout: post
title: "安装CentOS7.0后的系统配置及软件安装备忘"
description: ""
category: linux
tags: [linux, 系统运维]
---
{% include JB/setup %}

## 安装镜像获取

可从 https://www.centos.org/download/ 下载 CentOS 镜像文件。

## 镜像地址设置

可从 https://www.centos.org/download/mirrors/ 官网上查看有哪些可用的镜像地址。
下面以将镜像地址设置为阿里云镜像来说明设置步骤：

**step 1**. 备份

当镜像失效时，可重新使用原始镜像

``` shell
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
```

**step 2**. 下载新的CentOS-Base.repo 到/etc/yum.repos.d/

此处用的是阿里云镜像

``` shell
# CentOS 5
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-5.repo
# CentOS 6
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-6.repo
# CentOS 7
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
```

**step 3**. 之后运行 yum clean all && yum makecache 生成缓存

另外需要说明的是，很多系统镜像在安装完成后默认是没有开启网络功能的，我们需要配置并开启网络功能后才能执行此步操作。

## 网络配置和管理

### 配置IP地址

目录/etc/sysconfig/network-scripts下有名为ifcfg-eth0的脚本，它配置了网络接口eth0的各项属性

``` shell
HWADDR=00:0C:29:D4:D9:DD
TYPE=Ethernet
UUID=b9736563-22f5-42d5-9f41-9cd6c3e22683
ONBOOT=yes # 操作系统安装完成时，该选项默认为no，我们需要手工改为yes才能启用系统的网络功能
NM_CONTROLLED=yes
BOOTPROTO=static # 指定IP地址为静态IP(static)或动态IP(DHCP)
IPADDR=192.168.1.109 # IP地址(BOOTPROTO=static时有效)
NETMASK=255.255.255.0 # 子网掩码
GATEWAY=192.168.1.1 # 网关
DNS1=202.96.128.86
DNS2=202.96.134.33
```

### 配置DNS

/etc目录下有名为resolv.conf的脚本，由它来指定本机的DNS服务器

``` shell
; generated by /sbin/dhclient-script
nameserver 202.96.128.166
nameserver 202.96.134.133
```

在 dhcp 模式下我们不用关注该文件，因为会系统会自动帮我们找到合适的DNS域名服务器；
但在 static 模式下，我们就需要自己来维护该脚本了。

### 主机名称配置及IP映射

一般来说，我们可以通过文件 /etc/sysconfig/network 来永久地配置本机的主机名称，内容如下：

``` shell
NETWORKING=yes
HOSTNAME=centos66
```

不过在 CentOS7 中，主机名称的配置放到了 /etc/hostname 文件中。

在完成上面的配置之后，我们的主机名称除了能在命令提示符中显示之外，似乎没有其他用处。
我们希望主机名能替代具体的IP地址来使用，这就需要将主机名映射到具体的IP地址了。
主机在请求域名解析服务的时候会先查看本机的 /etc/hosts 文件，如果请求的域名在该文件中有匹配行，那就直接返回该匹配行中的IP地址。
/etc/hosts 文件中的内容形如：

``` shell
127.0.0.1   rhel
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
```

## 常用软件安装

可通过yum安装的软件：

``` shell
yum install -y net-tools # minimal模式下需要安装net-tools来安装常用网络管理工具
yum install -y bind # DNS服务
yum install -y traceroute
yum install -y tcpdump
yum install -y lsof
yum install -y git # 通过ssh-keygen -t rsa -b 4096 -C "your_email@example.com"生成key，然后添加到github配置中; ssh -T git@github.com可用于测试
yum install -y gcc
yum install -y gcc-c++
yum install -y clang
yum install -y emacs
yum install -y ncurses-devel
yum install -y samba # 远程文件传输服务
yum install -y pcre-devel openssl openssl-devel
yum install -y boost
yum install -y boost-devel
```

需源码安装的软件：

  + global-6.5 (https://ftp.gnu.org/pub/gnu/global/)
  + libevent2.x (https://github.com/nmathewson/Libevent/releases)
  + node-v8.2.0 (https://nodejs.org/en/download/releases/)
  + thrift-0.11.0 (https://github.com/ruleless/thrift/releases)
  + redis-3.0.6 (https://github.com/antirez/redis/releases)
  + shadowsocks-libev (https://github.com/ruleless/shadowsocks-libev/releases)
