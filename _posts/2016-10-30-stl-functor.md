---
layout: post
title: "STL 仿函数"
description: ""
category: stl
tags: [stl, functor]
---
{% include JB/setup %}

## 概要

仿函数的主要作用在哪里？

STL 提供的各种算法，往往有两个版本。
其中一个表现出最常用（或最直观）的某种运算；另一个则表现出泛化的运算流程，
允许用户以 template 参数来指定所要采用的策略。以 accumulate 为例，
其一般行为是将指定范围内的所有元素相加，泛化版本则允许你指定某种"操作"，
取代第一个版本中的"相加"行为。

据上所述，函数指针似乎能代替仿函数达到同样的目的，那又何必用所谓的仿函数呢？
原因在于函数指针毕竟不能满足 STL 对抽象性的要求，也不能满足软件积木的要求：
函数指针无法和 STL 的其他组件（如 Adapter）搭配，产生更灵活的变化。

## 仿函数基础类

stl 定义了两个基础的仿函数接口，它们定义了标准的接受一元或二元参数的仿函数内嵌型别。

``` c++
template <class _Arg, class _Result>
struct unary_function {
  typedef _Arg argument_type;
  typedef _Result result_type;
};

template <class _Arg1, class _Arg2, class _Result>
struct binary_function {
  typedef _Arg1 first_argument_type;
  typedef _Arg2 second_argument_type;
  typedef _Result result_type;
};
```

## 数值运算仿函数

``` c++
template <class _Tp>
struct plus : public binary_function<_Tp,_Tp,_Tp> {
  _Tp operator()(const _Tp& __x, const _Tp& __y) const { return __x + __y; }
};

template <class _Tp>
struct minus : public binary_function<_Tp,_Tp,_Tp> {
  _Tp operator()(const _Tp& __x, const _Tp& __y) const { return __x - __y; }
};

template <class _Tp>
struct multiplies : public binary_function<_Tp,_Tp,_Tp> {
  _Tp operator()(const _Tp& __x, const _Tp& __y) const { return __x * __y; }
};

template <class _Tp>
struct divides : public binary_function<_Tp,_Tp,_Tp> {
  _Tp operator()(const _Tp& __x, const _Tp& __y) const { return __x / __y; }
};

template <class _Tp>
struct modulus : public binary_function<_Tp,_Tp,_Tp>
{
  _Tp operator()(const _Tp& __x, const _Tp& __y) const { return __x % __y; }
};

template <class _Tp>
struct negate : public unary_function<_Tp,_Tp>
{
  _Tp operator()(const _Tp& __x) const { return -__x; }
};
```

## 关系运算仿函数

``` c++
template <class _Tp>
struct equal_to : public binary_function<_Tp,_Tp,bool>
{
  bool operator()(const _Tp& __x, const _Tp& __y) const { return __x == __y; }
};

template <class _Tp>
struct not_equal_to : public binary_function<_Tp,_Tp,bool>
{
  bool operator()(const _Tp& __x, const _Tp& __y) const { return __x != __y; }
};

template <class _Tp>
struct greater : public binary_function<_Tp,_Tp,bool>
{
  bool operator()(const _Tp& __x, const _Tp& __y) const { return __x > __y; }
};

template <class _Tp>
struct less : public binary_function<_Tp,_Tp,bool>
{
  bool operator()(const _Tp& __x, const _Tp& __y) const { return __x < __y; }
};

template <class _Tp>
struct greater_equal : public binary_function<_Tp,_Tp,bool>
{
  bool operator()(const _Tp& __x, const _Tp& __y) const { return __x >= __y; }
};

template <class _Tp>
struct less_equal : public binary_function<_Tp,_Tp,bool>
{
  bool operator()(const _Tp& __x, const _Tp& __y) const { return __x <= __y; }
};
```

## 逻辑运算仿函数

``` c++
template <class _Tp>
struct logical_and : public binary_function<_Tp,_Tp,bool>
{
  bool operator()(const _Tp& __x, const _Tp& __y) const { return __x && __y; }
};

template <class _Tp>
struct logical_or : public binary_function<_Tp,_Tp,bool>
{
  bool operator()(const _Tp& __x, const _Tp& __y) const { return __x || __y; }
};

template <class _Tp>
struct logical_not : public unary_function<_Tp,bool>
{
  bool operator()(const _Tp& __x) const { return !__x; }
};
```
