---
layout: post
title: "Unix I/O函数汇总"
description: ""
category: Unix环境编程
tags: [unix i/o]
---
{% include JB/setup %}

本节所列出的I/O函数围绕描述符工作，通常作为Unix内核中的系统调用实现。

## read和write函数

``` c++
#include <unistd.h>

// 成功返回读取的字节长度，失败返回-1
ssize_t read(int fd, void *buff, size_t len);

// 成功返回写入的字节长度，失败返回-1
ssize_t write(int fd, const void *buff, size_t len);
```

## readv和writev函数

``` c++
#include <sys/uio.h>

struct iovec
{
    void  *iov_base;
    size_t  iov_len;
};

ssize_t readv(int fd, const struct iovec *iov, int iovlen);
ssize_t writev(int ffd, const struct iovec *iov, int iovlen);
// 若成功则返回读入或写出的字节数，若出错则返回-1
```

这两个函数类似read和write，区别在于readv和writev允许单个系统调用读入到或写出自单个或多个缓冲区。
iovec结构数组中的元素的数目存在限制，具体取决于实现。BSD4.3和Linux均最多允许1024个。
POSIX要求在头文件<sys/uio.h>中定义IOV_MAX常值，其值最小为16。

## recvfrom和sendto函数

``` c++
#include <sys/socket.h>

// 成功返回接收到数据长度，失败返回-1
ssize_t recvfrom(int fd, void *buff, int len, struct sockaddr *addr, socklen_t *addrlen);

// 成功返回发送的数据长度，失败返回-1
ssize_t sendto(int fd, const void *buff, int len, const sockaddr *addr, socklen_t addrlen);
```

## recv和send函数

``` c++
#include <sys/socket.h>

// 成功返回接收的数据长度，失败返回-1
ssize_t recv(int fd, void *buff, size_t len, int flags);

// 成功返回发送的数据长度，失败返回-1
ssize_t send(int fd, const void *buff, size_t len, int flags);
```

## recvmsg和sendmsg函数

``` c++
#include <sys/socket.h>

struct msghdr
{
    void *msg_name; // 协议地址
    socklen_t msg_namelen; // 协议地址长度
    struct iovec *msg_iov; // 缓存数组
    int msg_iovlen; // 缓存数组大小
    void *msg_control; //  辅助数据
    socklen_t msg_controllen; // 辅助数据大小
    int msg_flags; // 内核通过该值向recvmsg返回标记
};

ssize_t recvmsg(int fd, struct msghdr *msg, int flags);
ssize_t sendmsg(int fd, struct msghdr *msg, int flags);
// 若成功返回读入或写出的字节数，失败返回-1
```

这两个函数是最通用的I/O函数。实际上我们可以把所有read、readv、recvfrom和recv替换成recvmsg调用。
类似地，各种输出函数调用也可以替换成sendmsg调用。

  + msg_name和msg_namelen用于套接字未连接的场合，例如未连接的UDP套接字。
  + msg_iov和msg_iovlen指定输入或输出缓冲区数组。
  + msg_control和msg_controllen指定可选的辅助数据的位置和大小。

msg_flags可用常值定义如下：

  + `MSG_DONTROUT`

    + 是否支持recv系列函数：否
    + 是否支持send系列函数：是
    + 是否可由内核返回：否

  + `MSG_DONTWAIT`

    + 是否支持recv系列函数：是
    + 是否支持send系列函数：是
    + 是否可由内核返回：否

  + `MSG_PEEK`

    + 是否支持recv系列函数：是
    + 是否支持send系列函数：否
    + 是否可由内核返回：否

  + `MSG_WAITALL`

    + 是否支持recv系列函数：是
    + 是否支持send系列函数：否
    + 是否可由内核返回：否

  + `MSG_EOR`

    本标志的返回条件是返回数据结束一个逻辑记录，TCP不使用本标志，因为它是一个字节流协议。

    + 是否支持recv系列函数：否
    + 是否支持send系列函数：是
    + 是否可由内核返回：是

  + `MSG_OOB`

    本标志绝不为TCP带外数据返回。它适用于OSI之类的其他协议族。

    + 是否支持recv系列函数：是
    + 是否支持send系列函数：是
    + 是否可由内核返回：是

  + `MSG_BCAST`

    本标志随BSD/OS引入，相对较新。它的返回条件是本数据报作为链路层广播收取或者其目的IP地址是一个广播地址。
    与IP_RECVDSTADDR套接字选项相比，本标志是用于判定一个UDP数据报是否发往某个广播地址的更好的方法。

    + 是否支持recv系列函数：否
    + 是否支持send系列函数：否
    + 是否可由内核返回：是

  + `MSG_MCAST`

    本标志随BSD/OS引入，相对较新。返回条件：本数据报作为链路层多播收取。

    + 是否支持recv系列函数：否
    + 是否支持send系列函数：否
    + 是否可由内核返回：是

  + `MSG_TRUNC`

    返回条件：本数据报被截断。也就是说，内核预备返回的数据超过进程事先分配的空间(所有iov_len成员之和)。

    + 是否支持recv系列函数：否
    + 是否支持send系列函数：否
    + 是否可由内核返回：是

  + `MSG_CTRUNC`

    返回条件：辅助数据被截断。也就是说，内核预备返回的辅助数据超过了进程事先分配的空间(msg_controllen)。

    + 是否支持recv系列函数：否
    + 是否支持send系列函数：否
    + 是否可由内核返回：是

  + `MSG_NOTIFICATION`

    本标志由SCTP接收者返回，指示读入的消息是一个事件通知，而不是数据消息。

    + 是否支持recv系列函数：否
    + 是否支持send系列函数：否
    + 是否可由内核返回：是

### 辅助数据

辅助数据可通过sendmsg和recvmsg函数，使用msghdr结构中的msg_control和msg_controllen这两个成员发送和接收。
辅助数据的结构定义如下：

``` c++
#incldue <sys/socket.h>

struct cmsghdr
{
    socklen_t cmsg_len; // 辅助数据长度
    int cmsg_level; // 协议
    int cmsg_type; // 类型
    ...            // cmsg_data
};
```

通过sendmsg和recvmsg函数可一次发送或接收一个或多个辅助数据对象，
下图展示了在一个控制缓冲区中出现2个辅助数据对象的例子。

![](/images/unix/io/io-cmsgpng.png)

msg_control指向第一个辅助数据对象，辅助数据的总长度则由msg_controllen指定。
每个对象的开头都是一个描述该对象的cmsghdr结构。
在cmsg_type成员和实际数据之间可以有填充字节，从数据结尾处到下一个辅助对象之间也可以有填充字节。

下述宏可以规避因填充字节而导致的程序复杂性：

``` c++
#include <sys/socket.h>
#include <sys/param.h>

// 直接返回pmsg->cmsg_control
struct cmsghdr *CMSG_FIRSTHDR(struct msghdr *pmsg);

// 返回指向下一个cmsghdr结构的指针，若不再有辅助数据对象则返回NULL
struct cmsghdr *CMSG_NXTHDR(struct msghdr *pmsg, struct cmsghdr *prevCMsg);

// 返回cmsghdr结构中的数据对象的首地址
unsigned char *CMSG_DATA(struct cmsghdr *cmsg);

// 返回给定数据量下存放到cmsg_len中的值
unsigned int CMSG_LEN(int length);

// 返回给定数据量下一个辅助数据对象总的大小
unsigned int CMSG_SPACE(int length);
```

上面讨论了辅助对象的结构和使用，那么辅助对象究竟是干吗的呢？
辅助数据也称控制信息，下表汇总了辅助数据的各种用途：

  + protocol:IPv4 cmsg_level:IPPROTO_IP

    cmsg_type取值区间：

    + `IP_RECVDSTADDR` 随UDP数据报接收目的地址
    + `IP_RECVIF` 随UDP数据报接收接口索引

  + protocol:IPv6 cmsg_level:OPPROTO_OPV6

    cmsg_type取值区间：

    + `IPV6_DSTOPTS` 指定/接收目的地选项
    + `IPV6_HOLIMIT` 指定/接收跳限
    + `IPV6_HOPOPTS` 指定/接收步跳选项
    + `IPV6_NEXTHOP` 指定下一跳地址
    + `IPV6_PKTINFO` 指定/接收分组信息
    + `IPV6_THDR` 指定/接收路由首部
    + `IPV6_TCLASS` 指定/接收分组流通类别

  + protocol:Unix域 cmsg_level:SOL_SOCKET

    cmsg_type取值区间：

    + `SCM_RIGHTS` 发送/接收描述符
    + `SCM_CREDS` 发送/接收用户凭证
