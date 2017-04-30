---
layout: post
title: "apache basic"
description: ""
category:
tags: []
---
{% include JB/setup %}

# apache模块编程

## 钩子

钩子实际上就是回调函数，apache 用一套统一的宏定义了统一的这样的回调函数的操作接口。
假设我们有钩子名 hookname ，并用 C 语言中的 ## 表示连接符，
那么，通过宏定义，apache 为该钩子定义的操作接口有：

  * `ap_hook_##hookname` 用于注册钩子。每类钩子都有与其对应的静态数组，此函数即是将钩子(回调函数)添加到该数组中。
  * `ap_run_##hookname` 执行所有注册了的钩子。
  * `ap_hook_get_##hookname` 获取钩子对应的静态数组。

例如，在 server/config.c 文件中有宏语句：

``` c++
AP_IMPLEMENT_HOOK_RUN_FIRST(int, handler, (request_rec *r),
                            (r), DECLINED)
```

那么，通过宏展开后，apache 为 handler 钩子定义的操作接口就是：

  * `ap_hook_handler`
  * `ap_run_handler`
  * `ap_hook_get_handler`

apache 中的钩子都是通过统一的宏定义的，但我们知道，它们的意义其实相差甚远。
这是如何做到的呢？答案就在调用钩子执行函数(即ap_run_##hookname)的时机。

## 处理http请求

apache是http服务器，http服务器最核心的业务当然是处理http请求。
那么，apache处理http请求的流程是怎样的呢？

![](/images/apache/basic/create-conn_rec.png)

上图在函数前用数字标记了其执行顺序，下面是对各函数功能的一个简单说明。

  1. 线程函数
  2. I/O事件监控。select仅仅监听监听套接字，这是因为apache对连接上的读写是按阻塞方式处理的
  3. 接入客户端连接。此步调用accept创建连接套接字
  4. 创建连接对象conn_rec。conn_rec保存了连接上下文，每个连接都需要创建一个该对象
  5. 处理连接
  6. 将连接分发到注册的钩子中进行处理。这里没有直接调用针对http协议的连接处理函数，主要是为了可扩展性考虑

上面的6个步骤主要描述了客户端连接的接入，但没有涉及接入后的请求处理。
在第6个步骤中，apache将连接分派给了我们事先注册的钩子。

在 modules/http/http_core.c 文件的 register_hooks 函数里，我们调用了
`ap_hook_process_connection(ap_process_http_connection, NULL, NULL, APR_HOOK_REALLY_LAST);`。
这表示我们为 process_connection 钩子注册了 ap_process_http_connection 函数，
该函数用于处理http请求。现在我们来看下http请求的处理流程：

![](/images/apache/basic/handle-request_rec.png)

各函数的主要说明如下：

  1. 承接上图第6步的函数调用
  2. ap_process_http_connection 是在 process_connection 钩子中注册的回调函数，主要用于处理http协议的连接
  3. 创建request_rec请求对象，并从连接中读取http协议头。每个request_rec对应一次http请求
  4. 请求处理函数
  5. 内部的请求处理函数
  6. 执行由 `ap_hook_translate_name` 注册的回调函数
  7. 执行由 `ap_hook_header_parser` 注册的回调函数
  8. 执行由 `ap_hook_access_checker` 注册的回调函数
  9. 执行由 `ap_hook_auth_checker` 注册的回调函数
  10. 执行由 `ap_hook_fixups` 注册的回调函数
  11. ap_invoke_handler
  12. 执行由 `ap_hook_insert_filter` 注册的回调函数
  13. 执行由 `ap_hook_handler` 注册的回调函数
