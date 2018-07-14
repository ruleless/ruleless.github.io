---
layout: post
title: "nginx分层模块设计"
description: ""
category: nginx
tags: [nginx]
---
{% include JB/setup %}

nginx的代码是以模块的形式组织起来的，当我们执行完configure后会生成ngx_modules.c，
这个文件中记录了nginx的所有模块（包括扩展模块）：

``` c
ngx_module_t *ngx_modules[] = {
    &ngx_core_module,
    &ngx_errlog_module,
    &ngx_conf_module,
    &ngx_regex_module,
    &ngx_events_module,
    &ngx_event_core_module,
    &ngx_epoll_module,
    &ngx_http_module,
    &ngx_http_core_module,
    &ngx_http_log_module,
    ...
}
```

我们先来看下nginx的模块结构图：

![](https://raw.githubusercontent.com/ruleless/ngx_port_map/master/doc/class.png)

  * `ngx_cycle_t`在程序启动时被创建，且是全局唯一的。
  * `ngx_cycle_t`中的`modules`是`ngx_module_t`数组，初始值从`ngx_modules`拷贝。
  * `ngx_module_t`关联一个或多个`ngx_command_t`对象，这个对象大有用处，一来它是创建次级模块的入口，二来它负责读取配置。
  * `ngx_*_module_t`是模块的上下文对象，它用于模块的个性化定制，一般来说，同一类别的模块拥有相同的上下文对象。
     如，`NGX_CORE_MODULE`类别的模块拥有`ngx_core_module_t`类型的上下文对象；`NGX_EVENT_MODULE`类别的模块拥有`ngx_event_module_t`类型的上下文对象。
  * `ngx_cytle_t`结构体中还有`conf_ctx`成员，它也是一个数组，初始化为跟`modules`一样的大小，`conf_ctx`用于存储模块的配置。
  
nginx的所有模块都是存放在`ngx_cycle_t`结构的`modules`数组中的，在内存中它们是线性的，但在逻辑上它们是分层的。
我们用下面的示意图展示模块的分层结构：

![](https://raw.githubusercontent.com/ruleless/ngx_port_map/master/doc/layer.png)

nginx中的模块根据类型字段（`ngx_module_t->type`）进行了分层、分组：

  * 顶层为`NGX_CORE_MODULE`类型的模块，它们的上下文对象为`ngx_core_module_t`类型。
    顶层模块可接入二层模块，一般通过`ngx_command_t`结构中的`set`方法来接入。
	如，`NGX_EVENT_MODULE`模块组就是通过`ngx_events_block`方法接入的。
