---
layout: post
title: "linux find命令使用介绍"
description: ""
category: linux
tags: []
---
{% include JB/setup %}

一个find命令的基本组成是：

``` shell
SYNOPSIS
    find [-H] [-L] [-P] [-D debugopts] [-Olevel] [path...] [expression]
```

可以将上述组成部分分为三块：参数、查找路径、表达式。

find命令的参数是比较少的，也容易理解，可以直接查看man；
查找路径也不必细说，就是find查找的目标目录；
而find的表达式则稍微复杂些，接下来讨论下find的表达式。

## 表达式定义

引用man手册页上的说明：

``` shell
EXPRESSIONS
    The  expression is made up of options (which affect overall operation rather than the processing of a
    specific file, and always return true), tests (which return a  true  or  false  value),  and  actions
    (which  have  side  effects  and  return a true or false value), all separated by operators.  -and is
    assumed where the operator is omitted.

    If the expression contains no actions other than -prune, -print is performed on all files  for  which
    the expression is true.
```

表达式由option、test或action组成，最简形式的option、test或action可看作原子表达式。
一个合法的表达式可递归定义如下：

  1. 原子表达式是合法表达式
  2. 若expr, expr1, expr2是合法表达式，则下列形式的表达式合法：
     1. \( expr \)
     2. ! expr
     3. expr1 expr2
     4. expr1 -a expr2
     5. expr1 -o expr2

`expr1 expr2` 和 `expr1 -a expr2` 表示逻辑与；
`expr1 -o expr2` 表示逻辑或。

find表达式类似逻辑表达式，且同大部分语言的逻辑表达式一样也有"短路"性质。
例如，`find . -name "test*" -a -mtime -2` 该条命令表示寻找当前目录下文件名前缀为test，
并且修改日期在两天之内的文件。"短路"是指，当find遍历到foo文件的时候，由于其不满足第一个条件，find不会对其进行第二条测试。

## 查找命令示例

我们抽取几个不好理解的命令来简要地说明下find的具体使用规则：

  + `find . -path "./foo"`

     表示查找 ./foo 目录，这条命令没有任何意义，列在这里只是为了跟下面的命令作个对比。

  + `find . -path "./foo" -prune`

     这条命令跟上条命令的执行结果是一样的。 `-prune` 总是校验为真，
     并且指示不进入使用了该动作(-prune属于action)的目录。

  + `find . -path "./foo" -prune -o -mtime -2"`

     查找当前目录下除 ./foo 目录及其下文件之外的所有修改时间在2天之内的文件。
     怎么理解这条命令呢？我们首先可以将该命令分解为三条表达式和两个逻辑联结词：
     `-path "./foo"` and `-prune` or `-mtime -2`。
     接着，我们分解下find的查找过程，当find遇到./foo文件夹的时候，
     `-path "./foo"` 条件校验为真，由于紧接其后的联结词为 and，所以接着测试 `-prune`，
     校验结果再次为真，而由于其后的联结词为 or，由于逻辑表达式的"短路"性质，
     于是，终止此次测试，另外 `-prune` 会跳过其测试的目录，所以 ./foo 目录会被find跳过。
     当find遍历到./test/tt文件时，`-path "./foo"`测试为假，由于其后的联结词为 and，
     所以跳过 `-prune` 测试，之后的联结词为 or，于是进行 `-mtime -2` 测试，
     `-mtime -2`测试为真则整个表达式结果为真，反之，则否。
