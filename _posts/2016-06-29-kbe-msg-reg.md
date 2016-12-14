---
layout: post
title: "kbengine消息处理函数注册"
description: ""
category: kbengine
tags: []
---
{% include JB/setup %}

在 kbengine 的任何一个服务器的目录下都存在几个含 interface 单词的文件，它们都跟消息处理相关。

以 baseappmgr 为例，我们进入 baseappmgr 的目录，输入 `ls *interface*`。会发现如下几个文件，
`baseappmgr_interface.cpp  baseappmgr_interface.h  baseappmgr_interface_macros.h`。

**baseappmgr_interface.h 的文件内容如**：

``` c++
...

/**
BASEAPPMGR所有消息接口在此定义
*/
NETWORK_INTERFACE_DECLARE_BEGIN(BaseappmgrInterface)

    ...

    // 某app主动请求look。
    BASEAPPMGR_MESSAGE_DECLARE_ARGS0(lookApp, NETWORK_FIXED_MESSAGE)

    // 某个app请求查看该app负载状态。
    BASEAPPMGR_MESSAGE_DECLARE_ARGS0(queryLoad, NETWORK_FIXED_MESSAGE)

    // 某个app向本app告知处于活动状态。
    BASEAPPMGR_MESSAGE_DECLARE_ARGS2(onAppActiveTick, NETWORK_FIXED_MESSAGE, COMPONENT_TYPE, componentType, COMPONENT_ID, componentID)

    // baseEntity请求创建在一个新的space中。
    BASEAPPMGR_MESSAGE_DECLARE_STREAM(reqCreateBaseAnywhere, NETWORK_VARIABLE_MESSAGE)

    ...

NETWORK_INTERFACE_DECLARE_END()
```

**baseappmgr_interface.cpp 的文件内容**：

``` c++
#include "baseappmgr_interface.h"
#define DEFINE_IN_INTERFACE
#define BASEAPPMGR
#include "baseappmgr_interface.h"

namespace KBEngine{
namespace BaseappmgrInterface{
}
}
```

仅看注释，我们就可猜测这些宏定义与消息处理相关。我们展开其中一个来具体分析。

&ensp;
---

我们先来看看头文件中 `*BEGIN` 和 `*END` 这对宏的定义：

``` c++
// 定义接口域名称
#ifndef DEFINE_IN_INTERFACE
#define NETWORK_INTERFACE_DECLARE_BEGIN(INAME) \
namespace INAME \
{ \
    extern Network::MessageHandlers messageHandlers; \

#else
#define NETWORK_INTERFACE_DECLARE_BEGIN(INAME) \
namespace INAME \
{ \
    Network::MessageHandlers messageHandlers; \

#endif

#define NETWORK_INTERFACE_DECLARE_END() }
```

因为我们只在源文件中定义了 `DEFINE_IN_INTERFACE` 宏，所以，

``` c++
NETWORK_INTERFACE_DECLARE_BEGIN(BaseappmgrInterface)

    ...

NETWORK_INTERFACE_DECLARE_END()
```

**在头文件和源文件中的展开形式分别是：**

``` c++
namespace BaseappmgrInterface
{
    extern Network::MessageHandlers messageHandlers;
}
```

**和**

``` c++
namespace BaseappmgrInterface
{
    Network::MessageHandlers messageHandlers;
}
```

&ensp;
---

我们再来观察下 “baseEntity请求创建在一个新的space中” 相关的消息宏展开。

`BASEAPPMGR_MESSAGE_DECLARE_STREAM` 相关的宏定义如下：

``` c++
#define BASEAPPMGR_MESSAGE_DECLARE_STREAM(NAME, MSG_LENGTH) \
    BASEAPPMGR_MESSAGE_HANDLER_STREAM(NAME) \
    NETWORK_MESSAGE_DECLARE_STREAM(Baseappmgr, NAME, NAME##BaseappmgrMessagehandler_stream, MSG_LENGTH)


#if defined(DEFINE_IN_INTERFACE)
#if defined(BASEAPPMGR)
#define BASEAPPMGR_MESSAGE_HANDLER_STREAM(NAME) \
    void NAME##BaseappmgrMessagehandler_stream::handle(Network::Channel* pChannel, KBEngine::MemoryStream& s) \
    { \
        KBEngine::Baseappmgr::getSingleton().NAME(pChannel, s); \
    } \
#else
#define BASEAPPMGR_MESSAGE_HANDLER_STREAM(NAME) \
    void NAME##BaseappmgrMessagehandler_stream::handle(Network::Channel* pChannel, KBEngine::MemoryStream& s) \
    { \
    } \
#endif
#else
#define BASEAPPMGR_MESSAGE_HANDLER_STREAM(NAME) \
    class NAME##BaseappmgrMessagehandler_stream : public Network::MessageHandler \
    { \
    public: \
        virtual void handle(Network::Channel* pChannel, KBEngine::MemoryStream& s); \
    };


#define NETWORK_MESSAGE_DECLARE_STREAM(DOMAIN, NAME, MSGHANDLER, MSG_LENGTH) \
    NETWORK_MESSAGE_HANDLER(DOMAIN, NAME, MSGHANDLER, MSG_LENGTH, _stream) \
    MESSAGE_STREAM(NAME)


#ifdef DEFINE_IN_INTERFACE
    #define NETWORK_MESSAGE_HANDLER(DOMAIN, NAME, HANDLER_TYPE, MSG_LENGTH, ARG_N) \
        HANDLER_TYPE* p##NAME = static_cast<HANDLER_TYPE*>(messageHandlers.add(#DOMAIN"::"#NAME,new NAME##Args##ARG_N, MSG_LENGTH, new HANDLER_TYPE)); \
        const HANDLER_TYPE& NAME = *p##NAME; \
#else
    #define NETWORK_MESSAGE_HANDLER(DOMAIN, NAME, HANDLER_TYPE, MSG_LENGTH, ARG_N) \
        extern const HANDLER_TYPE& NAME; \
#endif


#ifdef DEFINE_IN_INTERFACE
#define MESSAGE_STREAM(NAME)
#else
#define MESSAGE_STREAM(NAME) \
    class NAME##Args_stream : public Network::MessageArgs \
    { \
    public: \
        NAME##Args_stream():Network::MessageArgs(){} \
        ~NAME##Args_stream(){} \
        \
        virtual int32 dataSize(void) \
        { \
            return NETWORK_VARIABLE_MESSAGE; \
        } \
        virtual MessageArgs::MESSAGE_ARGS_TYPE type(void) \
        { \
            return MESSAGE_ARGS_TYPE_VARIABLE; \
        } \
        virtual void addToStream(MemoryStream& s) \
        { \
        } \
        virtual void createFromStream(MemoryStream& s) \
        { \
        } \
    }; \
#endif
```

所以，

``` c++
// baseEntity请求创建在一个新的space中。
BASEAPPMGR_MESSAGE_DECLARE_STREAM(reqCreateBaseAnywhere, NETWORK_VARIABLE_MESSAGE)
```

**在头文件和源文件中的展开形式分别是：**

``` c++
class reqCreateBaseAnywhereBaseappmgrMessagehandler_stream : public Network::MessageHandler
{
  public:
    virtual void handle(Network::Channel* pChannel, KBEngine::MemoryStream& s);
};

extern const reqCreateBaseAnywhereBaseappmgrMessagehandler_stream& reqCreateBaseAnywhere;

class reqCreateBaseAnywhereArgs_stream : public Network::MessageArgs
{
  public:
    reqCreateBaseAnywhereArgs_stream():Network::MessageArgs(){}
    ~reqCreateBaseAnywhereArgs_stream(){}

    virtual int32 dataSize(void)
    {
        return NETWORK_VARIABLE_MESSAGE;
    }
    virtual MessageArgs::MESSAGE_ARGS_TYPE type(void)
    {
        return MESSAGE_ARGS_TYPE_VARIABLE;
    }
    virtual void addToStream(MemoryStream& s)
    {
    }
    virtual void createFromStream(MemoryStream& s)
    {
    }
};
```

**和**

``` c++
void reqCreateBaseAnywhereBaseappmgrMessagehandler_stream::handle(Network::Channel* pChannel, KBEngine::MemoryStream& s)
{
    KBEngine::Baseappmgr::getSingleton().reqCreateBaseAnywhere(pChannel, s);
}

reqCreateBaseAnywhereBaseappmgrMessagehandler_stream* preqCreateBaseAnywhere =
    static_cast<reqCreateBaseAnywhereBaseappmgrMessagehandler_stream *>
    (messageHandlers.add(Baseappmgr::reqCreateBaseAnywhere,
        new reqCreateBaseAnywhereArgs_stream, NETWORK_VARIABLE_MESSAGE,
        new reqCreateBaseAnywhereBaseappmgrMessagehandler_stream));
const reqCreateBaseAnywhereBaseappmgrMessagehandler_stream& reqCreateBaseAnywhere = *preqCreateBaseAnywhere;
```
