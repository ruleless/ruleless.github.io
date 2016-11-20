---
layout: post
title: "套接字地址"
description: ""
category: Unix环境编程
tags: [bsd socket]
---
{% include JB/setup %}

## IPv4套接字地址结构

``` c++
struct in_addr {
    uint32_t s_addr;
};

struct sockaddr_in {
    unit8_t sin_len;
    uint8_t sin_family; // 地址族
    uint16_t sin_port; // 端口
    struct in_addr sin_addr; // 地址
    char sin_zero[8];
};
```

## 通用套接字地址结构

``` c++
struct sockaddr {
    uint8_t sa_len;
    uint8_t sa_family;
    char sa_data[14];
};
```

## IPv6套接字地址结构

``` c++
struct in6_addr {
    unit8_t s6_addr[16];
};

struct sockaddr_in6 {
    uint8_t sin6_len;
    uint8_t sin6_family;
    uint16_t sin6_port;
    uint32_t sin6_flowinfo;
    struct in6_addr sin6_addr;
    uint32_t sin6_scope_id;
};
```

## 兼容IPv6的通用套接字地址结构

``` c++
struct sockaddr_storage {
    unit8_t ss_len;
    uint8_t ss_family;
    ...
};
```

## 跟网络字节序相关的转换函数

``` c++
#include <netinet/in.h>

uint16_t htons(uint16_t hostval); // 主机序转换为网络序（短整型）
uint16_t ntohs(uint16_t netval);   // 网络序转换为主机序 （短整型）
uint32_t htonl(uint32_t hostval);  // 主机序转换为网络序（整型）
uint32_t ntohl(uint32_t netval);    // 网络序转换为主机序（短整型）
```

主机字节序检测程序：

``` c++
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

union ByteOrder {
	ushort n;
	char s[2];
};

int main(int argc, char *argv[])
{
	ByteOrder b;
	b.n = 0x0201;
	if (b.s[0] == 1)
		printf("小端序\n");
	else if (b.s[0] == 2)
		printf("大端序\n");
	else
		printf("Error\n");
	return 0;
}
```

## 字符串地址与socket地址之间的转换函数

``` c++
#include <netint/in.h>

//// IPv4地址转换函数

// 若字符串为有效地址，则返回1，否则返回0
int inet_aton(const char *strIp, struct in_addr *addr);

// 返回点分十进制字符串
char* inet_ntoa(struct in_addr addr);

// 若字符串为有效地址则返回32位的网络序IPv4地址，否则返回INADDR_NONE
uint32_t inet_addr(const char *strIp);

//// 通用地址转换函数

// 成功返回1，若字符串非有效表达式格式返回0，出错返回-1
int inet_pton(int family, const char *strIp, void *addr);

// 若成功则返回点分数字字串，失败返回NULL
const char* inet_ntop(int family, const void *addr, char *strIp, int strLen);

// 在inet_ntop中如何指定接收结果的字符串长度？
// 在<netinet/in.h>头文件中有如下定义：
#define INET_ADDRSTRLEN    16
#define INET6_ADDRSTRLEN   46
```
