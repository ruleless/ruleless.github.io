---
layout: post
title: "Linux epoll"
description: ""
category: Unix环境编程[Socket]
tags: [I/O]
---
{% include JB/setup %}

Linux I/O 多路复用技术在比较多的 TCP 网络服务器中有使用，即比较多的用到 select 函数。
Linux 2.6 内核中有提高网络 I/O 性能的新方法，即 epoll 。

## 为什么select落后？

首先，在Linux内核中，select 所用到的 FD_SET 是有限的，即内核中有个参数 __FD_SETSIZE 定义了每个 FD_SET 的句柄个数。
在我用的 2.6.15-25-386 内核中，该值是 1024，搜索内核源代码得到：

``` shell
// include/linux/posix_types.h:
#define __FD_SETSIZE         1024
```

也就是说，如果想要同时检测 1025 个句柄的可读状态是不可能用 select 实现的。
或者同时检测 1025 个句柄的可写状态也是不可能的。
其次，内核中实现 select 是使用轮询方法，即每次检测都会遍历所有 FD_SET 中的句柄，
显然，select 函数的执行时间与 FD_SET 中句柄的个数有一个比例关系，即 select 要检测的句柄数越多就会越费时。
当然，在前文中我并没有提及 poll 方法，事实上用 select 的朋友一定也试过 poll，
我个人觉得 select 和 poll 大同小异，个人偏好于用 select 而已。

## 内核中提高 I/O 性能的新方法 epoll

epoll 是什么？按照 man 手册的说法：是为处理大批量句柄而作了改进的 poll。
要使用 epoll 只需要以下的三个系统函数调用：`epoll_create， epoll_ctl， epoll_wait`。

先介绍 2 本书。
《The Linux Networking Architecture--Design and Implementation of Network Protocols in the Linux Kernel》，
以 2.4 内核讲解 Linux TCP/IP 实现，相当不错。
作为一个现实世界中的实现，很多时候你必须作很多权衡，这时候参考一个久经考验的系统更有实际意义。
举个例子，linux内核中 sk_buff 结构为了追求速度和安全，牺牲了部分内存，
所以在发送 TCP 包的时候，无论应用层数据多大，sk_buff 最小也有 272 的字节。
其实对于 socket 应用层程序来说，另外一本书《UNIX Network Programming Volume 1》意义更大一点。
2003年的时候，这本书出了最新的第3版本，不过主要还是修订第2版本。
其中第6章《I/O Multiplexing》是最重要的，Stevens给出了网络 I/O 的基本模型。
在这里最重要的莫过于 select 模型和 Asynchronous I/O模型。
从理论上说，AIO 似乎是最高效的，你的IO操作可以立即返回，然后等待 os 告诉你 I/O 操作完成。
但是一直以来，如何实现就没有一个完美的方案。
最著名的 windows 完成端口实现的 AIO，实际上也只是内部用线程池实现的罢了，
最后的结果是 I/O 有个线程池，你的应用程序也需要一个线程池。
很多文档其实已经指出了这引发的线程 context-switch 所带来的代价。
在 linux 平台上，关于网络 AIO 一直是改动最多的地方，2.4 的年代就有很多 AIO 内核 patch，最著名的应该算是 SGI。
但是一直到2.6内核发布，网络模块的 AIO 一直没有进入稳定内核版本
(大部分都是使用用户线程模拟方法，在使用了 NPTL 的 linux 上面其实和 windows 的完成端口基本上差不多了)。
2.6 内核所支持的 AIO 特指磁盘的AIO，支持 io_submit(), io_getevents() 以及对 Direct I/O 的支持(即：
就是绕过 VFS 系统 buffer 直接写硬盘，对于流服务器在内存平稳性上有相当的帮助)。

所以，剩下的 select 模型基本上就成为我们在 linux 上面的唯一选择。
其实，如果加上 no-block socket 的配置，可以完成一个"伪" AIO 的实现。
只不过推动力在于你而不是 os 而已。不过传统的 select/poll 函数有着一些无法忍受的缺点，
所以改进一直是 2.4-2.5 开发版本内核的任务，包括 /dev/poll，realtime signal 等等。
最终，Davide Libenzi 开发的 epoll 进入2.6内核成为正式的解决方案。

## epoll 的优点

  1. 支持一个进程打开大数目的 socket 描述符(fd)

	 select 最不能忍受的是一个进程所打开的 fd 是有一定限制的，由 FD_SETSIZE 设置，默认值是 2048。
	 对于那些需要支持上万连接数目的 IM 服务器来说显然太少了。
	 这时候你一是可以选择修改这个宏然后重新编译内核，不过资料也同时指出这样会带来网络效率的下降；
	 二是可以选择多进程的解决方案(传统的 Apache 方案)，不过虽然 linux 上面创建进程的代价比较小，但仍旧是不可忽视的，
	 加上进程间数据同步远比不上线程间同步高效，所以这也不是一种完美的方案。
	 不过 epoll 没有这个限制，它所支持的 fd 上限是最大可以打开文件的数目，这个数字一般远大于 select 所支持的2048。
	 举个例子，在 1GB 内存的机器上大约是 10 万左右，具体数目可以 cat /proc/sys/fs/file-max 察看，一般来说这个数目和系统内存关系很大。

  2. I/O效率不随 fd 数目增加而线性下降

     传统 select/poll 的另一个致命弱点就是当你拥有一个很大的 socket 集合，
	 由于网络得延时，使得任一时间只有部分的 socket 是"活跃" 的，
	 而 select/poll 每次调用都会线性扫描全部的集合，导致效率呈现线性下降。
	 但是 epoll 不存在这个问题，它只会对"活跃"的 socket 进行操作，
	 这是因为在内核实现中 epoll 是根据每个 fd 上面的 callback 函数实现的。
	 于是，只有"活跃"的 socket 才会主动去调用 callback 函数，其他 idle 状态的 socket 则不会，
	 在这点上，epoll实现了一个"伪" AIO，因为这时候推动力在 os 内核。
	 在一些 benchmark 中，如果所有的 socket 基本上都是活跃的，比如一个高速LAN环境，
	 epoll 也不比 select/poll 低多少效率，但若过多地调用 epoll_ctl，效率稍微有些下降。
	 然而一旦使用 idle connections 模拟 WAN 环境，那么 epoll 的效率就远在 select/poll 之上了。

  3. 使用 mmap 加速内核与用户空间的消息传递

     这点实际上涉及到 epoll 的具体实现。
	 无论是 select、poll 还是 epoll 都需要内核把 fd 消息通知给用户空间，
	 如何避免不必要的内存拷贝就显得很重要，在这点上，epoll 是通过内核于用户空间 mmap 同一块内存实现的。
	 而如果你像我一样从2.5内核就开始关注 epoll 的话，一定不会忘记手 工 mmap 这一步的。

  4. 内核微调

     这一点其实不算 epoll 的优点，而是整个 linux 平台的优点。
	 也许你可以怀疑 linux 平台，但是你无法回避 linux 平台赋予你微调内核的能力。
	 比如，内核 TCP/IP 协议栈使用内存池管理 sk_buff 结构，
	 可以在运行期间动态地调整这个内存 pool(skb_head_pool) 的大小
	 （通过 echo XXXX>/proc/sys/net/core/hot_list_length 来完成）。
	 再比如 listen 函数的第2个参数(TCP完成3次握手的数据包队列长度)，
	 也可以根据你平台内存大小来动态调整。
	 甚至可以在一个数据包面数目巨大但同时每个数据包本身大小却很小的特殊系统上尝试最新的 NAPI 网卡驱动架构。

## epoll 的工作模式

令人高兴的是，linux2.6 内核的 epoll 比其 2.5 开发版本的 /dev/epoll 简洁了许多，
所以，大部分情况下，强大的东西往往是简单的。唯一有点麻烦的是 epoll 有2种工作方式：LT 和 ET。

LT(level triggered) 是缺省的工作方式，并且同时支持 block 和 no-block socket。
在这种做法中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的 fd 进行 I/O 操作。
如果你不作任何操作，内核还是会继续通知你的，所以，这种模式编程出错误可能性要小一点。
传统的 select/poll 都是这种模型的代表。

ET(edge-triggered) 是高速工作方式，只支持 no-block socket。
在这种模式下，当描述符从未就绪变为就绪时，内核就通过 epoll 告诉你，
然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知，
直到你做了某些操作而导致那个文件描述符不再是就绪状态
(比如你在发送，接收或是接受请求，或者发送接收的数据少于一定量时导致了一个 EWOULDBLOCK 错误)。
但是请注意，如果一直不对这个 fd 作 I/O 操作(从而导致它再次变成未就绪)，
内核就不会发送更多的通知。不过在TCP协议中，ET 模式的加速效用仍需要更多的 benchmark 确认。

epoll 只有 epoll_create、epoll_ctl、epoll_wait 3个系统调用。

## epoll 所用到的数据结构

``` c++
typedef union epoll_data
{
    void *ptr;
    int fd;
    __uint32_t u32;
    __uint64_t u64;
} epoll_data_t;

struct epoll_event
{
    __uint32_t events; /* Epoll events */
    epoll_data_t data; /* User data variable */
};
```

结构体 epoll_event 被用于注册所感兴趣的事件和回传所发生待处理的事件，
而 epoll_data 联合体用来保存触发事件的某个文件描述符相关的数据。
例如一个 client 连接到服务器，服务器通过调用 accept 函数可以得到于这个 client 对应的 socket 文件描述符，
可以把这文件描述符赋给 epoll _data 的 fd 字段，以便后面的读写操作在这个文件描述符上进行。
epoll_event 结构体的 events 字段是表示感兴趣的事件和被触发的事件，可能的取值为：

  + `EPOLLIN` 表示对应的文件描述符可以读
  + `EPOLLOUT` 表示对应的文件描述符可以写
  + `EPOLLPRI` 表示对应的文件描述符有紧急的数据可读
  + `EPOLLERR` 表示对应的文件描述符发生错误
  + `EPOLLHUP` 表示对应的文件描述符被挂断

## epoll 所用到的函数

``` c++
int epoll_create(int size);
```

该函数生成一个epoll专用的文件描述符，其中的参数是指定生成描述符的最大范围。

``` c++
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

该函数用于控制某个文件描述符上的事件，可以注册事件，修改事件，删除事件。参数描述如下：

  + epfd 由 epoll_create 生成的 epoll 专用的文件描述符
  + op 要进行的操作，可能的取值 EPOLL_CTL_ADD 注册、EPOLL_CTL_MOD 修改、EPOLL_CTL_DEL 删除
  + fd 关联的文件描述符
  + event 指向 epoll_event 的指针

``` c++
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

该函数用于轮询 I/O 事件的发生。参数描述如下：

  + epfd 由epoll_create 生成的epoll专用的文件描述符
  + events 用于回传代处理事件的数组
  + maxevents 每次能处理的事件数
  + timeout 等待 I/O 事件发生的超时值

首先通过 create_epoll(maxfds) 来创建一个 epoll 的句柄，其中 maxfds 为你的 epoll 所支持的最大句柄数。
这个函数会返回一个新的 epoll 句柄，之后的所有操作都将通过这个句柄来进行操作。
在用完之后，记得用 close() 来关闭这个创建出来的 epoll 句柄。

之后在你的网络主循环里面，
调用 `epoll_wait(int epfd, epoll_event events, int max_events, int timeout)` 来查询所有的网络接口，
看哪一个可以读，哪一个可以写。基本的语法为：

``` c++
int nfds = epoll_wait(kdpfd, events, maxevents, -1);
```

其中 kdpfd 为用 epoll_create 创建之后的句柄，events是一个 epoll_event 的指针，
当 epoll_wait 函数操作成功之后，events 里面将储存所有的读写事件。
max_events 是当前需要监听的所有 socket 句柄数。
最后一个 timeout 参数指示 epoll_wait  的超时条件，为 0 时表示马上返回；
为 -1 时表示函数会一直等下去直到有事件返回；
为任意正整数时表示等这么长的时间，如果一直没有事件，则会返回。
一般情况下如果网络主循环是单线程的话，可以用 -1 来等待，
这样可以保证一些效率，如果是和主循环在同一个线程的话，则可以用 0 来保证主循环的效率。
epoll_wait 返回之后，应该进入一个循环，以便遍历所有的事件。

对epoll 的操作就这么简单，总共不过 4 个API：epoll_create, epoll_ctl, epoll_wait 和 close。
以下是 man 中的一个例子：

``` c++
struct epoll_event ev, *events;
for(;;)
{
    nfds = epoll_wait(kdpfd, events, maxevents, -1); //等待 I/O 事件
    for(n = 0; n < nfds; ++n)
    {
        if(events[n].data.fd == listener) // 如果是主 socket 的事件，则表示有新连接进入，需要进行新连接的处理。
        {
            client = accept(listener, (struct sockaddr *)&local,  &addrlen);
            if(client < 0)
            {
                perror("accept error");
                continue;
            }
            setnonblocking(client); // 将新连接置于非阻塞模式
            ev.events = EPOLLIN | EPOLLET;
            // 注意这里的参数 EPOLLIN | EPOLLET 并没有设置对写 socket 的监听，
            // 如果有写操作的话，这个时候 epoll 是不会返回事件的，
            // 如果要对写操作也监听的话，应该是 EPOLLIN | EPOLLOUT | EPOLLET。
            ev.data.fd = client; // 并且将新连接也加入 EPOLL 的监听队列
            if (epoll_ctl(kdpfd, EPOLL_CTL_ADD, client, &ev) < 0) // 设置好event之后，将这个新的event通过epoll_ctl
            {
                // 加入到 epoll 的监听队列里，
                // 这里用 EPOLL_CTL_ADD 来加一个新的 epoll 事件。
                // 可以通过 EPOLL_CTL_DEL 来减少一个 epoll 事件，
                // 通过 EPOLL_CTL_MOD 来改变一个事件的监听方式。
                fprintf(stderr, "epoll set insertion error: fd=%d", client);
                return -1;
            }
        }
        else // 如果不是主socket的事件的话，则代表这是一个用户的socket的事件，
        {
            do_use_fd(events[n].data.fd);
        }
    }
}
```

## Linux 下 epoll 编程实例

epoll 模型主要负责对大量并发用户的请求进行及时处理，完成服务器与客户端的数据交互。
其具体的实现步骤如下：

  1. 使用 epoll_create() 函数创建文件描述，设定可管理的最大 socket 描述符数目。
  2. 创建与 epoll 关联的接收线程，应用程序可以创建多个接收线程来处理 epoll 上的读通知事件，线程的数量依赖于程序的具体需要。
  3. 创建一个侦听 socket 的描述符 ListenSock，并将该描述符设定为非阻塞模式，
     调用 listen() 函数在该套接字上侦听有无新的连接请求，
	 在 epoll_event 结构中设置要处理的事件类型 EPOLLIN，
	 工作方式为 EPOLLET，以提高工作效率，同时使用 epoll_ctl() 来注册事件，
	 最后启动网络监视线程。
  4. 网络监视线程启动循环，epoll_wait() 等待 epoll 事件发生。
  5. 如果epoll事件表明有新的连接请求，则调用 accept() 函数，将用户 socket 描述符添加到 epoll_data 联合体，
     同时设定该描述符为非阻塞，并在 epoll_event 结构中设置要处理的事件类型为读和写，工作方式为 EPOLL_ET。
  6. 如果 epoll 事件表明 socket 描述符上有数据可读，则将该 socket 描述符加入可读队列，
     通知接收线程读入数据，并将接收到的数据放入到接收数据的链表中，经逻辑处理后，
	 将反馈的数据包放入到发送数据链表中，等待由发送线程发送。

例子代码：

``` c++
#include <iostream>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

#define MAXLINE 10
#define OPEN_MAX 100
#define LISTENQ 20
#define SERV_PORT 5555
#define INFTIM 1000

void setnonblocking(int sock)
{
    int opts;
    opts = fcntl(sock, F_GETFL);
    if(opts < 0)
    {
        perror("fcntl(sock,GETFL)");
        exit(1);
    }
    opts = opts | O_NONBLOCK;
    if(fcntl(sock, F_SETFL, opts) < 0)
    {
        perror("fcntl(sock,SETFL,opts)");
        exit(1);
    }
}

int main()
{
    int i, maxi, listenfd, connfd, sockfd, epfd, nfds;
    ssize_t n;
    char line[MAXLINE];
    socklen_t clilen;
    struct epoll_event ev,events[20]; //声明epoll_event结构体的变量, ev用于注册事件, events数组用于回传要处理的事件
    epfd=epoll_create(256); //生成用于处理accept的epoll专用的文件描述符, 指定生成描述符的最大范围为256
    struct sockaddr_in clientaddr;
    struct sockaddr_in serveraddr;
    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    setnonblocking(listenfd); //把用于监听的socket设置为非阻塞方式
    ev.data.fd=listenfd; //设置与要处理的事件相关的文件描述符
    ev.events=EPOLLIN | EPOLLET; //设置要处理的事件类型
    epoll_ctl(epfd,EPOLL_CTL_ADD,listenfd,&ev); //注册epoll事件
    bzero(&serveraddr, sizeof(serveraddr));
    serveraddr.sin_family = AF_INET;
    char *local_addr="200.200.200.204";
    inet_aton(local_addr,&(serveraddr.sin_addr));
    serveraddr.sin_port=htons(SERV_PORT);  //或者htons(SERV_PORT);
    bind(listenfd,(sockaddr *)&serveraddr, sizeof(serveraddr));
    listen(listenfd, LISTENQ);
    maxi = 0;

    for( ; ; )
    {
        nfds=epoll_wait(epfd,events,20,500); //等待epoll事件的发生
        for(i=0;i<nfds;++i) //处理所发生的所有事件
        {
            if(events[i].data.fd==listenfd)    /**监听事件**/
            {
                connfd = accept(listenfd,(sockaddr *)&clientaddr, &clilen);
                if(connfd<0)
                {
                    perror("connfd<0");
                    exit(1);
                }
                setnonblocking(connfd); //把客户端的socket设置为非阻塞方式
                char *str = inet_ntoa(clientaddr.sin_addr);
				printf("connect from %s\n", str);
                ev.data.fd=connfd; //设置用于读操作的文件描述符
                ev.events=EPOLLIN | EPOLLET; //设置用于注测的读操作事件
                epoll_ctl(epfd,EPOLL_CTL_ADD,connfd,&ev); //注册ev事件
            }
            else if(events[i].events&EPOLLIN)     /**读事件**/
            {
                if ( (sockfd = events[i].data.fd) < 0) continue;
                if ( (n = read(sockfd, line, MAXLINE)) < 0)
                {
                    if (errno == ECONNRESET)
                    {
                        close(sockfd);
                        events[i].data.fd = -1;
                    }
                    else
                    {
						printf("readline error\n");
                    }
                }
                else if (n == 0)
                {
                    close(sockfd);
                    events[i].data.fd = -1;
                }
                ev.data.fd=sockfd; //设置用于写操作的文件描述符
                ev.events=EPOLLOUT | EPOLLET; //设置用于注测的写操作事件
                epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev); //修改sockfd上要处理的事件为EPOLLOUT
            }
            else if(events[i].events&EPOLLOUT)    /**写事件**/
            {
                sockfd = events[i].data.fd;
                write(sockfd, line, n);
                ev.data.fd=sockfd; //设置用于读操作的文件描述符
                ev.events=EPOLLIN | EPOLLET; //设置用于注册的读操作事件
                epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev); //修改sockfd上要处理的事件为EPOLIN
            }
        }
    }
}
```
