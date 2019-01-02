---
layout: post
title: "nginx扩展模块编程"
description: ""
category: nginx
tags: [nginx]
---
{% include JB/setup %}

## hook阶段
在nginx http模块接收完客户端请求头部后，nginx立即会调用`ngx_http_core_run_phases`来执行框架定义的11个阶段。这11个阶段定义如下：
``` c++
typedef enum {
    NGX_HTTP_POST_READ_PHASE = 0,

    NGX_HTTP_SERVER_REWRITE_PHASE,

    NGX_HTTP_FIND_CONFIG_PHASE,
    NGX_HTTP_REWRITE_PHASE,
    NGX_HTTP_POST_REWRITE_PHASE,

    NGX_HTTP_PREACCESS_PHASE,

    NGX_HTTP_ACCESS_PHASE,
    NGX_HTTP_POST_ACCESS_PHASE,

    NGX_HTTP_PRECONTENT_PHASE,

    NGX_HTTP_CONTENT_PHASE,

    NGX_HTTP_LOG_PHASE
} ngx_http_phases;
```

各阶段含义如下：
  1. `NGX_HTTP_POST_READ_PHASE`： 第一个阶段，官方模块仅有realip模块hook该阶段
  2. `NGX_HTTP_SERVER_REWRITE_PHASE`：uri重写阶段，这个阶段的重写可以影响FIND_CONFIG阶段查找location块的结果
  3. `NGX_HTTP_FIND_CONFIG_PHASE`： 根据uri查找对应的location，并将location对应的handler设置到`r->content_handler`中。注意：**该阶段第三方模块不可hook**
  4. `NGX_HTTP_REWRITE_PHASE`： uri重写，这个阶段的重写在`FIND_CONFIG`阶段之后，所以框架不会根据这个阶段的重写结果查找location块
  5. `NGX_HTTP_POST_REWRITE_PHASE`：这个阶段用于避免循环重写uri，**第三方模块不可hook**
  6. `NGX_HTTP_PREACCESS_PHASE`：这个阶段跟POST_READ阶段使用的checker函数都是`ngx_http_core_generic_phase`，从名字上，我们可以理解它为预鉴权阶段
  7. `NGX_HTTP_ACCESS_PHASE`：鉴权阶段
  8. `NGX_HTTP_POST_ACCESS_PHASE`：鉴权结果反馈阶段，如果上一阶段鉴权失败，那该阶段会终止请求。注意：**该阶段第三方模块不可hook**
  9. `NGX_HTTP_PRECONTENT_PHASE`：官方模块仅有try_files模块hook了该阶段，从名字上看，该阶段表示预生成http包体，第三方模块一般难有类似需求，**所以一般不要hook该阶段**
  10. `NGX_HTTP_CONTENT_PHASE`：生成http包体阶段，第三方模块通常hook该阶段，该阶段有两种hook方式，我们在下面讨论
  11. `NGX_HTTP_LOG_PHASE`：该阶段在函数`ngx_http_free_request`中执行，http框架hook该阶段用于输出access log

## hook方式
第三方模块hook上面各阶段的方式基本类似，如rewrite module中的hook代码：
``` c++
static ngx_int_t
ngx_http_rewrite_init(ngx_conf_t *cf)
{
    ngx_http_handler_pt        *h;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    h = ngx_array_push(&cmcf->phases[NGX_HTTP_SERVER_REWRITE_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_rewrite_handler;

    h = ngx_array_push(&cmcf->phases[NGX_HTTP_REWRITE_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_rewrite_handler;

    return NGX_OK;
}
```

`NGX_HTTP_CONTENT_PHASE`阶段除了上面这种hook方式外，还有另一种比较常用的hook方式：
``` c++
static char *
ngx_http_dump(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_dump_loc_conf_t *dlcf = conf;
    ngx_http_core_loc_conf_t *clcf;
    char *rv;

    rv = ngx_conf_set_flag_slot(cf, cmd, conf);
    if (rv != NGX_CONF_OK) {
        return rv;
    }

    if (dlcf->dump_all) {
        clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
        clcf->handler = ngx_http_dump_hanlder;
    }

    return NGX_CONF_OK;
}
```
这第二种方式将hook函数设置到了`ngx_http_core_module`模块对应的location配置结构`ngx_http_core_loc_conf_t`的handler成员中。而handler将通过如下流程被调用：
  1. 在`NGX_HTTP_FIND_CONFIG_PHASE`阶段执行的`ngx_http_core_find_config_phase`函数中根据uri查找对应的location块，然后将location配置中的handler设置到`r->content_handler`
  2. 在`NGX_HTTP_CONTENT_PHASE`阶段执行的`ngx_http_core_content_phase`函数中先会判断`r->content_handler`是否为空，如果不为空就会执行该函数。这样，通过第二种方式设置的hook函数便被执行了。

另外，对于`NGX_HTTP_CONTENT_PHASE`阶段，还需要注意的是：第二种方式注册的hook函数会先于第一种方式注册的hook函数执行。而且，如果第二种方式注册的hook函数返回了`NGX_DECLINED`的以外的值，那么，第一种方式注册的hook将不会被执行；而如果返回`NGX_DECLINED`，则会依次执行第一种方式注册hook函数。

## 过滤器
nginx http模块有两条过滤链：http头部过滤链、http包体过滤链。这两条过滤链是以单链表的形式组织起来的，nginx为我们定义了单链表的头节点：
  1. 头部过滤链的头节点为：`ngx_http_top_header_filter`，它是一个全局的函数指针变量
  2. 包体过滤链的头节点为：`ngx_http_top_body_filter` ，它也是一个全局的函数指针变量

我们可以在nginx http过滤链中插入我们自己的过滤器，具体方式如下：
先在我们自己的模块中定义next节点变量：
``` c++
static ngx_http_output_header_filter_pt  ngx_http_next_header_filter;
static ngx_http_output_body_filter_pt    ngx_http_next_body_filter;
```

然后，在初始化函数中将我们自己的过滤器函数插入到过滤链头部：
``` c++
static ngx_int_t
ngx_http_dump_init(ngx_conf_t *cf)
{
    ngx_http_next_header_filter = ngx_http_top_header_filter;
    ngx_http_top_header_filter = ngx_http_dump_header_filter;

    ngx_http_next_body_filter = ngx_http_top_body_filter;
    ngx_http_top_body_filter = ngx_http_dump_body_filter;

    return NGX_OK;
}
```

过滤器函数一般形如：
``` c++
static ngx_int_t
ngx_http_dump_header_filter(ngx_http_request_t *r)
{
    return ngx_http_next_header_filter(r);
}

static ngx_int_t
ngx_http_dump_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    return ngx_http_next_body_filter(r, in);
}
```
