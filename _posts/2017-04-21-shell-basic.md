---
layout: post
title: "shell编程基础"
description: ""
category: shell
tags: []
---
{% include JB/setup %}

命令式的高级编程语言一般都离不开下面这些元素：

  1. 变量、运算符、表达式
  2. 控制流：顺序、条件、循环
  3. 模块化编程：函数、对象

## 变量、表达式

在shell脚本中，我们可以通过 `foo=value` 的方式来定义变量，通过 `$foo` 来引用它。
直接定义的变量的作用域是全局的，加上 **local** 修饰符则变为局部的，但我们只能在函数体内定义局部变量。

shell中的变量是没有类型的，或者我们可以认为shell中的所有变量都是字符串类型。
之所以这么设计，是因为shell本就为处理字符串而设计。

为编程方便，shell设计了如下变量运算符号：

  1. `${foo:-value}` 如果foo为空则返回value，否则返回foo
  2. `${foo:?"undefined"}` 如果foo为空就打印'undefined'并中断执行，否则返回foo
  3. `${foo:=value}` 如果foo为空就返回value并将foo置为value，否则返回foo
  4. `${foo:+value}` 如果foo为空则返回foo，否则返回value
  
又基于shell中所有变量都是字符串的特点，shell中为变量提供了如下运算符：

  1. `${foo:offset:len}` 截取foo从offset开始长度为len的字串
  2. `${foo#cut}` 从前往后匹配最短cut，并将其从foo中截取出去
  3. `${foo##cut}` 从前往后匹配最长cut，并将其从foo中截取出去
  4. `${foo%cut}` 从后往前匹配最短cut，并将其从foo中截取出去
  5. `${foo%%cut}` 从后往前匹配最长cut，并将其从foo中截取出去

## 控制流

## 函数
