---
layout: post
title: "C++临时对象"
description: ""
category: C++
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
另外，如果我们不希望发生此类型的隐式转换，可以在带单一形参的构造函数前加上关键字 `explicit`。

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

在 Linux 平台下，只出现了两次构造和析构函数的调用，这必然是我们所定义的 `o1` 和 `o2` 对象的创建和释放过程。
这就意味着没有创建临时对象，为什么呢？这必然是编译器优化的结果！
在 `testFunc1` 函数体中，我们定义了局部对象，且之后被作为结果返回。
但在返回之后，没有调用该局部变量的析构函数，这与 C++ 语言的作用域规则不符。
我们只能够认为函数体 `testFunc1` 中的局部对象 `o` 与上层调用堆栈中的变量 `o1` 合体了。

以最直观的C++语法知识来分析语句 `Obj o1 = testFunc1();`。
编译器应该先为右值创建临时对象，然后调用拷贝构造函数初始化 `o1` 。
但实际的运行结果却告诉我们在这个过程中编译器并没有创建临时对象。
抛开语法，就实际效果而言，编译器有必要创建临时对象来达到为 `o1` 赋值的目的吗？
如果，我们期望的是编译器默认的赋值行为，编译器没有必要创建临时对象，也没有必要调用拷贝构造函数。
但如果我们在拷贝构造函数中定义了独特的赋值行为，那一切另当别论。
