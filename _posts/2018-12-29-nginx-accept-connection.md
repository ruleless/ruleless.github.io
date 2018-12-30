---
layout: post
title: "nginx接入连接的过程"
description: ""
category: nginx
tags: [nginx]
---
{% include JB/setup %}

跟nginx连接接入相关的三个结构体为：
  1. `ngx_listening_t`: 监听结构体，需要为每个端口创建一个该结构体
  2. `ngx_connection_t`: 表示一个连接，对应套接字文件描述符，监听套接字也对应一个连接
  3. `ngx_event_t`: 连接上的事件，如：可读、可写、超时

`ngx_listening_t`以数组的方式保存在全局`ngx_cycle`变量的一个成员中：
``` c++
struct ngx_cycle_s {
    ...
    ngx_array_t               listening;
    ...
}
```

而listening数组的初始化分如下几步：
  1. 调用socket()创建套接字，随后调用listen()将套接字变为监听套接字，然后再调用bind()绑定监听地址。这些是在`ngx_open_listening_sockets`中完成的
  2. 关联`ngx_connection_t`结构，并设置好其读事件的回调。这一步在`ngx_event_core_module`的`ngx_event_process_init`函数中完成

`ngx_event_process_init`中初始化listening数组的代码如下：
``` c++
static ngx_int_t
ngx_event_process_init(ngx_cycle_t *cycle)
{
    ...

    /* for each listening socket */
    ls = cycle->listening.elts;
    for (i = 0; i < cycle->listening.nelts; i++) {
        c = ngx_get_connection(ls[i].fd, cycle->log);
        c->type = ls[i].type;
        c->log = &ls[i].log;
        c->listening = &ls[i];
        ls[i].connection = c;

        rev = c->read;
        rev->log = c->log;
        rev->accept = 1;

        if (!(ngx_event_flags & NGX_USE_IOCP_EVENT)) {
            if (ls[i].previous) {
                /*
                 * delete the old accept events that were bound to
                 * the old cycle read events array
                 */
                old = ls[i].previous->connection;
                if (ngx_del_event(old->read, NGX_READ_EVENT, NGX_CLOSE_EVENT)
                    == NGX_ERROR)
                {
                    return NGX_ERROR;
                }
                old->fd = (ngx_socket_t) -1;
            }
        }

        rev->handler = (c->type == SOCK_STREAM) ? ngx_event_accept
                                                : ngx_event_recvmsg;
        if (ngx_use_accept_mutex) {
            continue;
        }
        if (ngx_add_event(rev, NGX_READ_EVENT, 0) == NGX_ERROR) {
            return NGX_ERROR;
        }
    }
    ...
}
```

在这个函数中，监听套接字的可读事件回调被设置为了**ngx_event_accept**；
并且在`ngx_use_accept_mutex`为0的情况下注册了监听套接字的可读事件。
当监听套接字的可读事件被注册后，下一步就是准备接入连接了；
那么如果`ngx_use_accept_mutex`为1呢？当该值为1时，多个子进程需要先抢到互斥锁，才能准备接入连接。

子进程的抢锁逻辑在`ngx_process_events_and_timers`函数中：
``` c++
void
ngx_process_events_and_timers(ngx_cycle_t *cycle)
{
    ngx_uint_t  flags;
    ngx_msec_t  timer, delta;

    if (ngx_timer_resolution) {
        timer = NGX_TIMER_INFINITE;
        flags = 0;
    } else {
        timer = ngx_event_find_timer();
        flags = NGX_UPDATE_TIME;
    }

    if (ngx_use_accept_mutex) {
        if (ngx_accept_disabled > 0) {
            ngx_accept_disabled--;
        } else {
            if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {
                return;
            }

            if (ngx_accept_mutex_held) {
                flags |= NGX_POST_EVENTS;
            } else {
                if (timer == NGX_TIMER_INFINITE
                    || timer > ngx_accept_mutex_delay)
                {
                    timer = ngx_accept_mutex_delay;
                }
            }
        }
    }

    delta = ngx_current_msec;
    (void) ngx_process_events(cycle, timer, flags);
    delta = ngx_current_msec - delta;

    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
    ngx_event_process_posted(cycle, &ngx_posted_accept_events);

    if (ngx_accept_mutex_held) {
        ngx_shmtx_unlock(&ngx_accept_mutex);
    }

    if (delta) {
        ngx_event_expire_timers();
    }

    ngx_event_process_posted(cycle, &ngx_posted_events);
}
```

当`ngx_accept_disabled<=0`时，子进程才会去抢锁，如果抢锁成功，就会注册监听套接字的可读事件。
`ngx_accept_disabled`初始化为0，当有连接接入，nginx调用`ngx_event_accept`时`ngx_accept_disabled`会被设置为：
``` c++
ngx_accept_disabled = ngx_cycle->connection_n / 8 - ngx_cycle->free_connection_n;
```

为方便理解，我们将`ngx_cycle->free_connection_n`用`ngx_cycle->connection_n - current_connection_n`替换，于是有：
``` c++
ngx_accept_disabled = current_connection_n - 7*ngx_cycle->connection_n / 8;
```

**也就是说当现有连接数大于总连接数的八分之七时，新连接会优先让渡给其他子进程处理**。
通过这一机制，nginx能保证在高负载情况下，各子进程处理的连接数基本相近，不会出现子进程负载不均衡的情况。

前面说过，监听套接字的可读事件回调被设置为了**ngx_event_accept**，当有新连接接入时，就会触发该函数的调用。
**ngx_event_accept**做了如下事情：
  1. 调用accept()创建连接套接字
  2. 设置：`ngx_accept_disabled = ngx_cycle->connection_n / 8 - ngx_cycle->free_connection_n;`
  3. 调用`ngx_get_connection`获取连接对象`ngx_connection_t`
  4. 调用ngx_listening_t的handler，参数就是第三步的`ngx_connection_t`对象

通过第4步对`ngx_listening_t`的handler调用，新接入的连接将被派发到各业务模块处理，
如http模块的`ngx_http_init_connection`、mail模块的`ngx_mail_init_connection`、stream模块的`ngx_stream_init_connection`。
