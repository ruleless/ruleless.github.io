---
layout: post
title: "Nginx网络I/O事件注册及回调"
description: ""
category: nginx
tags: [nginx]
---
{% include JB/setup %}

Nginx 是事件驱动式设计，跟大部分服务器程序一样，它主要处理两类事件：
**网络事件** 和 **定时器事件**。
网络事件来自操作系统内核，而定时器事件则是在程序内部实现订阅和分发。

Nginx 中跟网络事件相关的结构体有：`ngx_listening_t`, `ngx_connection_t`, `ngx_event_t`。
从命名，我们便可大致猜测各结构体的作用，大部分网络程序的I/O部分多多少少都有上述三者的影子。

`ngx_listening_t` 存储于 `ngx_cycle_t` 中的 `listening` 成员中。
而该成员将在 `ngx_init_cycle` 调用中被初始化。如下：

``` c++
ngx_cycle_t *
ngx_init_cycle(ngx_cycle_t *old_cycle)
{
	...

	n = old_cycle->listening.nelts ? old_cycle->listening.nelts : 10;

    cycle->listening.elts = ngx_pcalloc(pool, n * sizeof(ngx_listening_t));
    if (cycle->listening.elts == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    cycle->listening.nelts = 0;
    cycle->listening.size = sizeof(ngx_listening_t);
    cycle->listening.nalloc = n;
    cycle->listening.pool = pool;

	...

	if (ngx_open_listening_sockets(cycle) != NGX_OK) {
        goto failed;
    }

	...
}
```

`ngx_open_listening_sockets` 干了些什么事情呢？
该函数会为 `cycle->listening` 数组中的每个监听地址打开一个监听套接字。
但从上面列出的代码来看，`cycle->listening` 数组被初始化为了0个元素，
`cycle->listening` 又是在何时增加的元素呢？
在 Nginx 中，为 `cycle->listening` 添加元素的函数只有 `ngx_create_listening`。
通过查找，我们可以知道，在 http, mail 等模块调用了 `ngx_create_listening`。
就 http 模块而言，它会在什么时候调用 `ngx_create_listening` 呢？在解析配置的时候。
而各配置块的解析是在 `cycle->listening` 初始化之后，`ngx_open_listening_sockets` 调用之前完成的。

在执行完 `ngx_init_cycle` 之后，我们就有了一组 `ngx_listening_t` 对象，它存储在 `cycle->listening` 数组中。
但在此时，我们调用的 Socket Api 仅有 `socket/bind/listen/setsockopt`，
如我们使用epoll，注册网络I/O事件的 `epoll_ctl` 并未被调用。
所以，仅有 `ngx_listening_t` 是不能构建整套事件机制的，甚至连外部主机连入事件都处理不了。
