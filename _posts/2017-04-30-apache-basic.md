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

要进行apache扩展模块编程，首先离不开的就是钩子。钩子实际上就是回调函数。
apache有一套跟钩子相关的宏定义：

``` c++
APR_DECLARE_EXTERNAL_HOOK /* 用于头文件，起声明作用 */

APR_HOOK_STRUCT/APR_HOOK_LINK /* 静态全局变量定义 */
APR_IMPLEMENT_EXTERNAL_HOOK_VOID /* 钩子执行函数实现(总是调用所有注册的钩子) */
APR_IMPLEMENT_EXTERNAL_HOOK_RUN_ALL /* 钩子执行函数实现(根据返回值尝试调用所有注册的钩子) */
APR_IMPLEMENT_EXTERNAL_HOOK_RUN_FIRST /* 钩子执行函数实现(只调用第一个) */
```

假设我们有钩子名为hookname，并用C语言中的##表示连接符，那么，通过宏定义，apache为该钩子定义的操作接口有：

  * `ap_hook_##hookname` 用于注册钩子。每类钩子都有与其对应的静态数组，此函数即是将钩子(回调函数)添加到该数组中。
  * `ap_run_##hookname` 执行所有注册了的钩子。
  * `ap_hook_get_##hookname` 获取钩子对应的静态数组。

据上所述，通过apache宏定义的钩子有变量，有操作接口，所以我们可以将钩子看作是面向对象模型中的对象。
示意图如下：

![](/images/apache/basic/hook-class.png)

举个apache中的实际例子，在server/config.c文件中有宏语句：

``` c++
AP_IMPLEMENT_HOOK_RUN_FIRST(int, handler, (request_rec *r),
                            (r), DECLINED)
```

那么，通过宏展开后，apache为handler钩子定义的操作接口是：

  * `ap_hook_handler`
  * `ap_run_handler`
  * `ap_hook_get_handler`

示意图如下：

![](/images/apache/basic/hook-class-handler.png)

我们进行扩展模块编程主要用到的是ap_hook_\*函数，表示注册钩子；
而，ap_run_\*函数主要是在apache中调用的，不同的调用时机以及调用参数赋予了钩子不同的含义。

我们调用ap_hook_\*注册钩子时，应注意些什么呢？
首先是钩子的类型，即注册的钩子会在哪个阶段被调用；其次是钩子执行的顺序，即同类型的钩子执行的先后关系。
apache中常用的钩子如下：

![](/images/apache/basic/hook-class-all.png)

除了上图中提到的不同的钩子类型，
在编写apache扩展模块的时候，我们通常还需要考虑到注册的同类钩子的执行顺序。
apache为我们定义了如下的宏来控制相同类型的钩子的执行顺序：

``` c++
/** run this hook first, before ANYTHING */
#define APR_HOOK_REALLY_FIRST   (-10)
/** run this hook first */
#define APR_HOOK_FIRST          0
/** run this hook somewhere */
#define APR_HOOK_MIDDLE         10
/** run this hook after every other hook which is defined*/
#define APR_HOOK_LAST           20
/** run this hook last, after EVERYTHING */
#define APR_HOOK_REALLY_LAST    30
```

我们之前说过，每种类型的钩子都会关联一个静态数组，在注册完钩子之后，apache会为该数组重新排序，
而排序的依据就是我们注册钩子时传进去的执行顺序参数。排好序的钩子数组在内存中的布局如下：

![](/images/apache/basic/hook-order.png)

## http请求处理

apache是http服务器，http服务器最核心的业务当然是处理http请求。
那么，apache处理http请求的流程是怎样的呢？

![](/images/apache/basic/create-conn_rec.png)

上图在函数前用数字标记了其执行顺序，下面是对各函数功能的一个简单说明。

  1. 线程函数
  2. I/O事件监控。select仅仅监控监听套接字，这是因为apache对连接上的读写是按阻塞方式处理的
  3. 接入客户端连接。此步调用accept创建连接套接字
  4. 创建连接对象conn_rec。conn_rec保存了连接上下文，每个连接都需要创建一个该对象
  5. 处理连接
  6. 将连接分发到注册的钩子中进行处理。这里没有直接调用针对http协议的连接处理函数，主要是为了可扩展性考虑

上面的6个步骤主要描述了客户端连接的接入，但没有涉及接入后的请求处理。
在第6个步骤中，apache将连接分派给了我们事先注册的钩子。

在modules/http/http_core.c文件的register_hooks函数里，我们调用了
`ap_hook_process_connection(ap_process_http_connection, NULL, NULL, APR_HOOK_REALLY_LAST);`。
这表示我们在process_connection钩子中注册了ap_process_http_connection函数，
该函数用于处理http请求。现在我们来看下http请求的处理流程：

![](/images/apache/basic/handle-request_rec.png)

各函数的主要说明如下：

  1. 承接上图第6步的函数调用
  2. ap_process_http_connection是在process_connection钩子中注册的回调函数，主要用于处理http协议的连接
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

概要的说，apache处理http请求的流程就是：
首先接入客户端连接，创建conn_rec结构体；然后再从连接中读入请求，创建request_rec结构体；
再之后，通过apache的hook机制将请求分发到其他模块进行处理。
之所以不在apache核心代码里面直接处理请求的原因是，http协议是复杂的，不同的http请求可能对应着迥异的处理逻辑。
例如，有些http请求直接返回本地文件到客户端即可，有些却需要向内部服务器进行二次请求(反向代理)。

上面简单描述了http请求的处理过程，但没有涉及到怎么向客户端回复http响应。
下面来讲讲apache怎么向客户端发送http响应。

## 过滤器

apache用过滤器来处理http响应，过滤器结构定义如下：

``` c++
struct ap_filter_t {
    ap_filter_rec_t *frec;
    void *ctx;
    ap_filter_t *next;
    
    request_rec *r;
    conn_rec *c;
};
```

结构体里面定义了next指针，过滤器以单链表的形式组成了过滤链。
存储连接的conn_rec和存储请求的request_rec结构体都关联了过滤链，
