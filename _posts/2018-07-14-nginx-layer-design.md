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
  * 二层`NGX_EVENT_MODULE`模块组是通过顶层模块`ngx_events_module`接入的；
    二层`NGX_HTTP_MODULE`模块组是通过顶层模块`ngx_http_module`接入的。
	具体是怎么接入的我们在下面讨论。

上面的分层模块图有些模块是加粗的，这些模块是组织模块，负责本模块组的管理，未加粗的则是实际干活的功能模块。

我们接下来看看，二层模块是如何从顶层模块接入的：

![](https://raw.githubusercontent.com/ruleless/ngx_port_map/master/doc/control-flow.png)

`ngx_preinit_modules`负责所有模块初始化：

``` c
ngx_int_t ngx_preinit_modules()
{
    ngx_uint_t  i;

    ngx_max_module = 0;
    for (i = 0; ngx_modules[i]; i++) {
        ngx_modules[i]->index = i;
        ngx_modules[i]->name = ngx_module_names[i];
    }

    ngx_modules_n = i;
    ngx_max_module = ngx_modules_n + NGX_MAX_DYNAMIC_MODULES;

    return NGX_OK;
}
```

`cycle->modules[i]->ctx->create_conf`和`cycle->modules[i]->ctx->init_conf`只针对顶层模块，用来创建和初始化各模块的配置。
`ngx_conf_parse`读取并解析配置文件，解析完配置后调用`ngx_conf_handler`。
`ngx_conf_handler`遍历模块的命令数组，然后根据配置项与命令参数一一对比，如果符合，则调用命令结构的set函数。
注意，**set函数就是接入二层模块的关键**。

对于不负责接入的功能模块，set函数一般置为设置配置的函数。
比如，`ngx_errlog_module`的`commands`定义为：

``` c
static ngx_command_t  ngx_errlog_commands[] = {
    {ngx_string("error_log"),
     NGX_MAIN_CONF|NGX_CONF_1MORE,
     ngx_error_log,
     0,
     0,
     NULL},

    ngx_null_command
};
```

`ngx_error_log`定义如下：

``` c
static char *ngx_error_log(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_log_t  *dummy;
    dummy = &cf->cycle->new_log;
    return ngx_log_set_log(cf, &dummy);
}
```

而对于需要接入二层模块的顶层模块，主要就是依赖set函数来完成二层模块的接入了。
我们来看下`ngx_events_module`的`commands`成员的定义：

``` c
static ngx_command_t  ngx_events_commands[] = {
    { ngx_string("events"),
      NGX_MAIN_CONF|NGX_CONF_BLOCK|NGX_CONF_NOARGS,
      ngx_events_block,
      0,
      0,
      NULL },

      ngx_null_command
};
```

`NGX_EVENT_MODULE`模块组就是在`ngx_events_block`函数中接入的。
为了把二层模块的接入过程阐释清楚，我们完整的展示下`ngx_events_block`函数的定义。
所有的二层模块的接入过程基本类似，我们自己写nginx的扩展的时候，最好也参照现有方法。

``` c
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

函数调用流程如下：

![](https://raw.githubusercontent.com/ruleless/ngx_port_map/master/doc/control-flow-1.png)

是不是跟`ngx_init_cycle`中初始化顶层模块的过程很像？
`NGX_EVENT_MODULE`模块组的初始化过程也是三步：
  1. 调用上下文对象中的`create_conf`方法创建配置
  2. 调用`ngx_conf_parse`加载事件模块配置
  3. 调用上下文对象中的`init_conf`完成初始化

不过需要注意的是，事件模块中的上下文对象是`ngx_event_module_t`结构；
而顶层模块中的上下文对象是`ngx_core_module_t`结构，虽然它们拥有相同的配置创建和初始化函数，但它们是来自不同的结构体。

最后，我们通过下面的示意图来展示下`ngx_cycle_t`结构中的模块数组和配置数组，以及二级模块的配置存放方式：

![](https://raw.githubusercontent.com/ruleless/ngx_port_map/master/doc/modules.png)
