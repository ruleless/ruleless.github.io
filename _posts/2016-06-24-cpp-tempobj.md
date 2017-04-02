---
layout: post
title: "C++临时对象"
description: ""
category: C/C++
tags: []
---
{% include JB/setup %}

在以下情况下，编译器可能会创建临时对象：

  1. 发生隐式类型转换
  2. 按传值方式调用函数
  3. 按值返回

为演示上述三种情况，我们构建如下示例类：

``` c++
class Obj
{
  public:
    Obj(const char *desc)
    {
        strncpy(mDesc, desc, 256);
        printf("default construct %p \"%s\"\n", (void *)this, mDesc);
    }

    Obj(const Obj &o)
    {
        strncpy(mDesc, o.mDesc, 256);
        printf("copy construct %p \"%s\"\n", (void *)this, mDesc);
    }

    Obj& operator= (const Obj &o)
    {
        strncpy(mDesc, o.mDesc, 256);
        printf("operator= %p \"%s\"\n", (void *)this, mDesc);
        return *this;
    }

    virtual ~Obj()
    {
        printf("~ destruct %p \"%s\"\n", (void *)this, mDesc);
    }
  private:
    char mDesc[256];
};
```

## 发生隐式类型转换

例：

``` c++
void testFunc1(const Obj &i)
{
}
```

我们以 `testFunc1("test situation 1")` 的形式调用 `testFunc1`，此时即会发生隐式类型转换。
编译器会以 `"test situation 1"` 为实参构建一个临时对象。
如果我们不希望发生此类型的隐式转换，可以在带单一形参的构造函数前加上关键字 `explicit`。

## 按传值方式调用函数

例：

``` c++
void testFunc1(Obj i)
{
}
```

我们以如下方式调用上述函数：

``` c++
Obj o("situation 2");
testFunc1(o);
```

运行程序我们得到如下结果：

``` shell
default construct 0x7fff20933ff0 "situation 2"
copy construct 0x7fff20933ee8 "situation 2"
~ destruct 0x7fff20933ee8 "situation 2"
~ destruct 0x7fff20933ff0 "situation 2"
```

我们发现，构造了两个对象，很明显，其中一个是临时对象。
编译器以 `o` 为参数调用 `Obj` 的拷贝构造函数构造了一个临时对象。

## 按值返回

例：

``` c++
Obj testFunc1()
{
    Obj o("test1");
    return o;
}

Obj testFunc2()
{
    return Obj("test2"); // it's called RVO(Return Value Optimization)
}
```

以如下方式调用上述函数：

``` c++
Obj o1 = testFunc1();
Obj o2 = testFunc2();
```

我们在 Linux 2.6.32-431.el6.x86_64 下的运行结果为：

``` shell
default construct 0x7ffff3c58060 "test1"
default construct 0x7ffff3c57f58 "test2"
~ destruct 0x7ffff3c57f58 "test2"
~ destruct 0x7ffff3c58060 "test1"
```

在 vs2005 下的运行结果为：

``` shell
default construct 0018F8C4 "test1"
copy construct 0018FC04 "test1"
~ destruct 0018F8C4 "test1"
default construct 0018FAFC "test2"
~ destruct 0018FAFC "test2"
~ destruct 0018FC04 "test1"
```

以最直观的C++语法知识来分析语句 `Obj o1 = testFunc1();`，
编译器应该先为右值创建临时对象，然后调用拷贝构造函数初始化 `o1` 。
但实际的运行结果却告诉我们在这个过程中编译器并没有创建临时对象。
事实上，我们应该纠正上述观念，在语句 `Obj o1 = testFunc1();` 中，编译器不会为右值创建临时对象，更不会调用拷贝构造函数来初始化左值。
`testFunc1` 返回的临时对象将直接拷贝给左值，左值将延续该对象的生命，虽然该对象是在 `testFunc1` 函数体中初始化的。

但无论如何， `testFunc1` 中的对象 `o` 在 `testFunc1` 的生命周期中应该要有创建和释放的过程。
而在 Linux 平台下我们没有看到该过程，唯一的解释是：
GNU C 编译器将 `o` 拷贝给了上层堆栈的 `o1`，并让 `o1` 延续了其生命周期。
vs2005 平台下，在 `testFunc1` 的生命周期中，我们看到了 `o` 的创建和释放过程，
并且，在 `o` 返回时，编译器调用拷贝构造函数初始化了一个临时对象。
