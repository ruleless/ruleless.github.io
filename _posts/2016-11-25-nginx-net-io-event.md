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

## ngx_listening_t

`ngx_listening_t` 存储于 `ngx_cycle_t` 中的 `listening` 成员中。
而该成员将在 `ngx_init_cycle` 调用中被初始化。如下：

``` c++
ngx_cycle_t *
ngx_init_cycle(ngx_cycle_t *old_cycle)
{
    ngx_uint_t           i, n;

	// ...

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

	// ...

    if (ngx_open_listening_sockets(cycle) != NGX_OK) {
        goto failed;
    }

	// ...

    return NULL;
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

## Nginx中的epoll模块接入

### 事件模块

在初始化了cycle->listening后，我们需要通过epoll(或select, kqueue, iocp等)来为监听套接字设置读写回调
(监听套接字的可读事件表示有主机连入)。
在Nginx中，epoll是以模块的形式接入进来的。
epoll 在 Nginx 中的模块变量为 `ngx_epoll_module`，它属于事件模块。

Nginx 采用了分层模块结构设计，顶层模块由 Nginx 自身管理。比如，
在 `ngx_init_cycle` 有一段代码为：

``` c++
ngx_cycle_t *
ngx_init_cycle(ngx_cycle_t *old_cycle)
{
    ngx_uint_t           i, n;

	// ...

    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->type != NGX_CORE_MODULE) {
            continue;
        }

        module = cycle->modules[i]->ctx;

        if (module->create_conf) {
            rv = module->create_conf(cycle);
            if (rv == NULL) {
                ngx_destroy_pool(pool);
                return NULL;
            }
            cycle->conf_ctx[cycle->modules[i]->index] = rv;
        }
    }

	// ...

    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->type != NGX_CORE_MODULE) {
            continue;
        }

        module = cycle->modules[i]->ctx;

        if (module->init_conf) {
            if (module->init_conf(cycle,
                                  cycle->conf_ctx[cycle->modules[i]->index])
                == NGX_CONF_ERROR)
            {
                environ = senv;
                ngx_destroy_cycle_pools(&conf);
                return NULL;
            }
        }
    }

    // ...

    return NULL;
}
```

类型为 NGX_CORE_MODULE 的模块的 `create_conf` 和 `init_conf` 会被调用。
类型为 NGX_CORE_MODULE 的摸块即属于 Nginx 中的顶层模块。
顶层模块除负责像日志打印、配置解析等核心业务外，也需要管理二级模块的接入。
实际上，上层模块负责下层模块的接入，这是一种很自然的设计。
假如，我们自己在 Nginx 中扩展了二级模块，而由于业务复杂，我们需要进一步进行模块划分。
而新划分出的模块则属于三级模块，那这三级模块的接入不由我们自己定义的二级模块接入又该由谁负责呢？

事件模块的类型为 NGX_EVENT_MODULE，若以 epoll 为事件驱动的话，Nginx 中与事件相关的模块有：
`ngx_events_module`, `ngx_event_core_module`, `ngx_epoll_module`。
其中 ngx_events_module 属于顶层模块，但它负责二级事件模块的接入；
ngx_event_core_module 是核心的事件模块，它负责各平台具体的事件模块的管理；
ngx_epoll_module 是Linux平台下高效的事件驱动模块。
我们来看下这三个模块的定义：

``` c++
ngx_module_t  ngx_events_module = {
    NGX_MODULE_V1,
    &ngx_events_module_ctx,                /* module context */
    ngx_events_commands,                   /* module directives */
    NGX_CORE_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};

ngx_module_t  ngx_event_core_module = {
    NGX_MODULE_V1,
    &ngx_event_core_module_ctx,            /* module context */
    ngx_event_core_commands,               /* module directives */
    NGX_EVENT_MODULE,                      /* module type */
    NULL,                                  /* init master */
    ngx_event_module_init,                 /* init module */
    ngx_event_process_init,                /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};

ngx_module_t  ngx_poll_module = {
    NGX_MODULE_V1,
    &ngx_poll_module_ctx,                  /* module context */
    NULL,                                  /* module directives */
    NGX_EVENT_MODULE,                      /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};
```

### 事件模块配置块的解析

虽然必要的事件模块的初始化工作是在 `ngx_event_core_module` 模块的 `ngx_event_process_init` 函数完成的，
但若不在 nginx.conf 配置文件中加入下面的配置块，那么事件模块是不能正常工作的。

``` shell
...

events {
    worker_connections  1024;
}

...
```

我们来看事件模块针对该条配置的解析：

``` c++
static ngx_command_t  ngx_events_commands[] = {
    { ngx_string("events"),
      NGX_MAIN_CONF|NGX_CONF_BLOCK|NGX_CONF_NOARGS,
      ngx_events_block,
      0,
      0,
      NULL },

      ngx_null_command
};

static char *
ngx_events_block(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    char                 *rv;
    void               ***ctx;
    ngx_uint_t            i;
    ngx_conf_t            pcf;
    ngx_event_module_t   *m;

    if (*(void **) conf) {
        return "is duplicate";
    }

    /* count the number of the event modules and set up their indices */

    ngx_event_max_module = ngx_count_modules(cf->cycle, NGX_EVENT_MODULE);

    ctx = ngx_pcalloc(cf->pool, sizeof(void *));
    if (ctx == NULL) {
        return NGX_CONF_ERROR;
    }

    *ctx = ngx_pcalloc(cf->pool, ngx_event_max_module * sizeof(void *));
    if (*ctx == NULL) {
        return NGX_CONF_ERROR;
    }

    *(void **) conf = ctx;

    for (i = 0; cf->cycle->modules[i]; i++) {
        if (cf->cycle->modules[i]->type != NGX_EVENT_MODULE) {
            continue;
        }

        m = cf->cycle->modules[i]->ctx;

        if (m->create_conf) {
            (*ctx)[cf->cycle->modules[i]->ctx_index] =
                                                     m->create_conf(cf->cycle);
            if ((*ctx)[cf->cycle->modules[i]->ctx_index] == NULL) {
                return NGX_CONF_ERROR;
            }
        }
    }

    pcf = *cf;
    cf->ctx = ctx;
    cf->module_type = NGX_EVENT_MODULE;
    cf->cmd_type = NGX_EVENT_CONF;

    rv = ngx_conf_parse(cf, NULL);

    *cf = pcf;

    if (rv != NGX_CONF_OK) {
        return rv;
    }

    for (i = 0; cf->cycle->modules[i]; i++) {
        if (cf->cycle->modules[i]->type != NGX_EVENT_MODULE) {
            continue;
        }

        m = cf->cycle->modules[i]->ctx;

        if (m->init_conf) {
            rv = m->init_conf(cf->cycle,
                              (*ctx)[cf->cycle->modules[i]->ctx_index]);
            if (rv != NGX_CONF_OK) {
                return rv;
            }
        }
    }

    return NGX_CONF_OK;
}
```

nginx.conf 配置文件中的 events 块就是在 `ngx_events_block` 函数中完成解析的。
该函数会执行以下事情：

  1. 统计事件模块的个数，保存到 `ngx_event_max_module` 全局变量中
  2. 为 `cycle->conf_ctx[ngx_event_module.index]` 分配一个有 `ngx_event_max_module` 个元素的指针数组
  3. 针对每个事件模块调用其 `create_conf` 以创建配置项空间，并将返回的指针保存到第2步创建的指针数组中
  4. 调用 `ngx_conf_parse` 解析 events 配置块
  5. 调用每个事件模块的 `init_conf` 方法

### 事件模块的初始化

`ngx_events_block` 函数统计了事件模块的个数，完成了事件模块相关配置项的解析。
但事件相关的初始化工作并不是在这里进行的，而是在 `ngx_event_process_init` 函数中完成的。
该函数是 `ngx_event_core_module` 的 `init_process` 成员，那么模块的 `init_process` 函数一般在什么时候调用呢？
在 master 启动方式中它是在如下代码中被调用的：

``` c++
static void
ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data)
{
    ngx_int_t worker = (intptr_t) data;

    ngx_process = NGX_PROCESS_WORKER;
    ngx_worker = worker;

    ngx_worker_process_init(cycle, worker);

    ngx_setproctitle("worker process");

    // ...
}

static void
ngx_worker_process_init(ngx_cycle_t *cycle, ngx_int_t worker)
{
    ngx_int_t         n;
    ngx_uint_t        i;

	// ...

    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->init_process) {
            if (cycle->modules[i]->init_process(cycle) == NGX_ERROR) {
                /* fatal */
                exit(2);
            }
        }
    }

	// ...
}
```

其中，ngx_worker_process_cycle函数是在子进程中被调用的。
那么，ngx_event_process_init干了什么呢？

``` c++
static ngx_int_t
ngx_event_process_init(ngx_cycle_t *cycle)
{
    ngx_uint_t           m, i;
    ngx_event_t         *rev, *wev;
    ngx_listening_t     *ls;
    ngx_connection_t    *c, *next, *old;
    ngx_core_conf_t     *ccf;
    ngx_event_conf_t    *ecf;
    ngx_event_module_t  *module;

    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
    ecf = ngx_event_get_conf(cycle->conf_ctx, ngx_event_core_module);

    if (ccf->master && ccf->worker_processes > 1 && ecf->accept_mutex) {
        ngx_use_accept_mutex = 1;
        ngx_accept_mutex_held = 0;
        ngx_accept_mutex_delay = ecf->accept_mutex_delay;

    } else {
        ngx_use_accept_mutex = 0;
    }

    ngx_queue_init(&ngx_posted_accept_events);
    ngx_queue_init(&ngx_posted_events);

    if (ngx_event_timer_init(cycle->log) == NGX_ERROR) {
        return NGX_ERROR;
    }

    for (m = 0; cycle->modules[m]; m++) {
        if (cycle->modules[m]->type != NGX_EVENT_MODULE) {
            continue;
        }

        if (cycle->modules[m]->ctx_index != ecf->use) {
            continue;
        }

        module = cycle->modules[m]->ctx;

        if (module->actions.init(cycle, ngx_timer_resolution) != NGX_OK) {
            /* fatal */
            exit(2);
        }

        break;
    }

	// ...

    cycle->connections =
        ngx_alloc(sizeof(ngx_connection_t) * cycle->connection_n, cycle->log);
    if (cycle->connections == NULL) {
        return NGX_ERROR;
    }

    c = cycle->connections;

    cycle->read_events = ngx_alloc(sizeof(ngx_event_t) * cycle->connection_n,
                                   cycle->log);
    if (cycle->read_events == NULL) {
        return NGX_ERROR;
    }

    rev = cycle->read_events;
    for (i = 0; i < cycle->connection_n; i++) {
        rev[i].closed = 1;
        rev[i].instance = 1;
    }

    cycle->write_events = ngx_alloc(sizeof(ngx_event_t) * cycle->connection_n,
                                    cycle->log);
    if (cycle->write_events == NULL) {
        return NGX_ERROR;
    }

    wev = cycle->write_events;
    for (i = 0; i < cycle->connection_n; i++) {
        wev[i].closed = 1;
    }

    i = cycle->connection_n;
    next = NULL;

    do {
        i--;

        c[i].data = next;
        c[i].read = &cycle->read_events[i];
        c[i].write = &cycle->write_events[i];
        c[i].fd = (ngx_socket_t) -1;

        next = &c[i];
    } while (i);

    cycle->free_connections = next;
    cycle->free_connection_n = cycle->connection_n;

    /* for each listening socket */
    ls = cycle->listening.elts;
    for (i = 0; i < cycle->listening.nelts; i++) {

#if (NGX_HAVE_REUSEPORT)
        if (ls[i].reuseport && ls[i].worker != ngx_worker) {
            continue;
        }
#endif

        c = ngx_get_connection(ls[i].fd, cycle->log);

        if (c == NULL) {
            return NGX_ERROR;
        }

        c->log = &ls[i].log;

        c->listening = &ls[i];
        ls[i].connection = c;

        rev = c->read;

        rev->log = c->log;
        rev->accept = 1;

#if (NGX_HAVE_DEFERRED_ACCEPT)
        rev->deferred_accept = ls[i].deferred_accept;
#endif

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

        rev->handler = ngx_event_accept;

        if (ngx_use_accept_mutex
#if (NGX_HAVE_REUSEPORT)
            && !ls[i].reuseport
#endif
           )
        {
            continue;
        }

        if (ngx_add_event(rev, NGX_READ_EVENT, 0) == NGX_ERROR) {
            return NGX_ERROR;
        }
#endif
    }

    return NGX_OK;
}
```

ngx_event_process_init函数完成了如下几件事情：

  1. 定时器初始化
  2. 调用每个事件模块的init函数
  3. 预分配 cycle->connections, cycle->read_events, cycle->write_evets
  4. 为 cycle->listening 中的每个监听套接字分配一个 ngx_connection_t 对象
  5. 将 cycle->listening 中监听套接字的可读事件回调设置为 `ngx_event_accept`，并为其注册可读事件

在第2步，如果我们使用epoll，则会调用epoll模块的init方法。

在第5步，我们为 cycle->listening 中的监听套接字设置了可读事件回调方法。
从此之后，我们便可以处理外部主机连入请求了。
每当有外部主机连入时，ngx_event_accept便会被调用，
该函数会通过accept系统调用获取到连接套接字，并为该连接套接字设置可读/可写事件回调。

至于如何注册可读写事件回调，及可读写事件的回调机制？
我们来看下 epoll 模块提供的事件操作接口：

``` c++
ngx_event_module_t  ngx_epoll_module_ctx = {
    &epoll_name,
    ngx_epoll_create_conf,               /* create configuration */
    ngx_epoll_init_conf,                 /* init configuration */

    {
        ngx_epoll_add_event,             /* add an event */
        ngx_epoll_del_event,             /* delete an event */
        ngx_epoll_add_event,             /* enable an event */
        ngx_epoll_del_event,             /* disable an event */
        ngx_epoll_add_connection,        /* add an connection */
        ngx_epoll_del_connection,        /* delete an connection */
#if (NGX_HAVE_EVENTFD)
        ngx_epoll_notify,                /* trigger a notify */
#else
        NULL,                            /* trigger a notify */
#endif
        ngx_epoll_process_events,        /* process the events */
        ngx_epoll_init,                  /* init the events */
        ngx_epoll_done,                  /* done the events */
    }
};

ngx_module_t  ngx_epoll_module = {
    NGX_MODULE_V1,
    &ngx_epoll_module_ctx,               /* module context */
    ngx_epoll_commands,                  /* module directives */
    NGX_EVENT_MODULE,                    /* module type */
    NULL,                                /* init master */
    NULL,                                /* init module */
    NULL,                                /* init process */
    NULL,                                /* init thread */
    NULL,                                /* exit thread */
    NULL,                                /* exit process */
    NULL,                                /* exit master */
    NGX_MODULE_V1_PADDING
};
```

epoll 模块定义了I/O可读/可写事件的操作接口：

  + `ngx_epoll_add_event` 注册可读/可写事件
  + `ngx_epoll_del_event` 反注册可读/可写事件
  + `ngx_epoll_add_connection` 同ngx_epoll_add_event，只不过可以同时注册可读和可写事件
  + `ngx_epoll_del_connection` 反注册可读和可写事件
  + `ngx_epoll_process_events` 事件分发
  + `ngx_epoll_init` 初始化
  + `ngx_epoll_done` 去初始化

## ngx_connection_t, ngx_event_t

到目前为止，我们讲到了事件的初始化，并且看到了epoll向外提供的操作接口。
如果我们熟悉epoll的使用，并且概览一遍上述epoll模块提供的接口的实现代码，
很快便能明白其中的工作原理。我们接下来讲解用户注册可读/可写事件的方式。

在上面我们说过可以通过ngx_epoll_add_event和ngx_epoll_add_connection注册读写事件。
不过，既然Nginx是跨平台的，用户便不能直接用这种方式了，Nginx定义了如下事件操作宏：

``` c++
#define ngx_process_events   ngx_event_actions.process_events
#define ngx_done_events      ngx_event_actions.done

#define ngx_add_event        ngx_event_actions.add
#define ngx_del_event        ngx_event_actions.del
#define ngx_add_conn         ngx_event_actions.add_conn
#define ngx_del_conn         ngx_event_actions.del_conn
```

其中，ngx_event_actions会在ngx_epoll_init初始化为上述的epoll模块(如果我们使用epoll)的接口。
如下：

``` c++
static ngx_int_t
ngx_epoll_init(ngx_cycle_t *cycle, ngx_msec_t timer)
{
    ngx_epoll_conf_t  *epcf;

    epcf = ngx_event_get_conf(cycle->conf_ctx, ngx_epoll_module);

    if (ep == -1) {
        ep = epoll_create(cycle->connection_n / 2);

        if (ep == -1) {
            ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                          "epoll_create() failed");
            return NGX_ERROR;
        }

#if (NGX_HAVE_EVENTFD)
        if (ngx_epoll_notify_init(cycle->log) != NGX_OK) {
            ngx_epoll_module_ctx.actions.notify = NULL;
        }
#endif

#if (NGX_HAVE_FILE_AIO)

        ngx_epoll_aio_init(cycle, epcf);

#endif
    }

    if (nevents < epcf->events) {
        if (event_list) {
            ngx_free(event_list);
        }

        event_list = ngx_alloc(sizeof(struct epoll_event) * epcf->events,
                               cycle->log);
        if (event_list == NULL) {
            return NGX_ERROR;
        }
    }

    nevents = epcf->events;

    ngx_io = ngx_os_io;

    ngx_event_actions = ngx_epoll_module_ctx.actions;

#if (NGX_HAVE_CLEAR_EVENT)
    ngx_event_flags = NGX_USE_CLEAR_EVENT
#else
    ngx_event_flags = NGX_USE_LEVEL_EVENT
#endif
                      |NGX_USE_GREEDY_EVENT
                      |NGX_USE_EPOLL_EVENT;

    return NGX_OK;
}
```

说明了读写事件注册的接口之后，我们还剩最后一个问题。用户怎么向epoll提供自定义的回调接口？
为了搞清楚这个问题，我们先直接来看下epoll模块中事件分发的相关代码：

``` c++
static ngx_int_t
ngx_epoll_process_events(ngx_cycle_t *cycle, ngx_msec_t timer, ngx_uint_t flags)
{
    int                events;
    uint32_t           revents;
    ngx_int_t          instance, i;
    ngx_uint_t         level;
    ngx_err_t          err;
    ngx_event_t       *rev, *wev;
    ngx_queue_t       *queue;
    ngx_connection_t  *c;

    events = epoll_wait(ep, event_list, (int) nevents, timer);

	// ...

    if (events == 0) {
        if (timer != NGX_TIMER_INFINITE) {
            return NGX_OK;
        }

        ngx_log_error(NGX_LOG_ALERT, cycle->log, 0,
                      "epoll_wait() returned no events without timeout");
        return NGX_ERROR;
    }

    for (i = 0; i < events; i++) {
        c = event_list[i].data.ptr;

        instance = (uintptr_t) c & 1;
        c = (ngx_connection_t *) ((uintptr_t) c & (uintptr_t) ~1);

        rev = c->read;

        if (c->fd == -1 || rev->instance != instance) {
            ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                           "epoll: stale event %p", c);
            continue;
        }

        revents = event_list[i].events;

        ngx_log_debug3(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                       "epoll: fd:%d ev:%04XD d:%p",
                       c->fd, revents, event_list[i].data.ptr);

        if (revents & (EPOLLERR|EPOLLHUP)) {
            ngx_log_debug2(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                           "epoll_wait() error on fd:%d ev:%04XD",
                           c->fd, revents);
        }

        if ((revents & (EPOLLERR|EPOLLHUP))
             && (revents & (EPOLLIN|EPOLLOUT)) == 0)
        {
            revents |= EPOLLIN|EPOLLOUT;
        }

        if ((revents & EPOLLIN) && rev->active) {

#if (NGX_HAVE_EPOLLRDHUP)
            if (revents & EPOLLRDHUP) {
                rev->pending_eof = 1;
            }
#endif

            rev->ready = 1;

            if (flags & NGX_POST_EVENTS) {
                queue = rev->accept ? &ngx_posted_accept_events
                                    : &ngx_posted_events;

                ngx_post_event(rev, queue);

            } else {
                rev->handler(rev);
            }
        }

        wev = c->write;

        if ((revents & EPOLLOUT) && wev->active) {

            if (c->fd == -1 || wev->instance != instance) {
                ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                               "epoll: stale event %p", c);
                continue;
            }

            wev->ready = 1;

            if (flags & NGX_POST_EVENTS) {
                ngx_post_event(wev, &ngx_posted_events);

            } else {
                wev->handler(wev);
            }
        }
    }

    return NGX_OK;
}
```

当读事件准备就绪时会调用 `rev->handler(rev)`；
当写事件准备就绪时会调用 `wev->handler(wev)`。
ngx_event_t 的定义如下：

``` c++
typedef struct {
    void            *data;
    unsigned         write:1;
    unsigned         accept:1;
    /* used to detect the stale events in kqueue and epoll */
    unsigned         instance:1;
    /*
     * the event was passed or would be passed to a kernel;
     * in aio mode - operation was posted.
     */
    unsigned         active:1;
    unsigned         disabled:1;
    /* the ready event; in aio mode 0 means that no operation can be posted */
    unsigned         ready:1;
    unsigned         oneshot:1;
    /* aio operation is complete */
    unsigned         complete:1;
    unsigned         eof:1;
    unsigned         error:1;
    unsigned         timedout:1;
    unsigned         timer_set:1;
    unsigned         delayed:1;
    unsigned         deferred_accept:1;
    /* the pending eof reported by kqueue, epoll or in aio chain operation */
    unsigned         pending_eof:1;
    unsigned         posted:1;
    unsigned         closed:1;
    /* to test on worker exit */
    unsigned         channel:1;
    unsigned         resolver:1;
    unsigned         cancelable:1;

    unsigned         available:1;

    ngx_event_handler_pt  handler;

    ngx_uint_t       index;
    ngx_log_t       *log;
    ngx_rbtree_node_t   timer;
    /* the posted queue */
    ngx_queue_t      queue;
} ngx_event_t;
```

从以上论述可知，用户可通过 handler 成员来定义自己的读写事件回调方法。
