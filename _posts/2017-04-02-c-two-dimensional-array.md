---
layout: post
title: "C语言中使用二维数组的另一种方式"
description: ""
category: C/C++
tags: []
---
{% include JB/setup %}

## C语言中的二维数组

C语言是支持二维数组的。例如我们定义 `int array[16][16]` ，
那么我们就能够通过 `array[i][j]` 的方式访问到二维数组中的元素。

对于二维数组的这种使用方式，哪怕是初学者也能很快理解和掌握。
但对于下面一种情况，则不那么直观了。

假如，我们事先不知道二维数组的大小，我们需要在运行时动态分配二维数组。
那需要怎么做呢？我如果很长一段时间没用C的话，就会不自觉的写成下面这种形式：

``` c++
int **array;
array = (int **)malloc(m*n*sizeof(int));
array[i][j] = 1;
```

当然，上面这种情况会比较容易看出问题所在，更隐晦的一种写法像下面这样：

``` c++
int static_array[16][16], **array;
if (m <= 16 && n <= 16)
    array = static_array;
else
    array = (int **)malloc(m*n*sizeof(int));

array[i][j] = 1;
```

类似上面的这种代码，我的意图是在数组太大的时候动态分配我所需要的二维数组。
但实际上，这段代码是不能够按照我的预期工作的。

对于 `static_array` ，如果我们写 `static_array[i][j]` ，
编译器会将我们的访问地址转换为 `static_array+i*16+j` 。
二维数组实际也是分配到一块连续区域上的，C语言通过指针运算达到模拟二维数组的效果。
而对于 `int **array` ，变量被定义为了指向指针的指针，这与二维数组不相干的。
如果我们写 `array[i][j]` 编译器并不会帮我们把地址映射到 `array+i*16+j` ，而是 `*(array+i)+j` 。

所以，我们如果要使用动态分配的二维数组，我们应该要使用如下代码：

``` c++
int **array;
array = (int **)malloc(m*sizeof(int *));
for (int i = 0; i < n, ++i)
    array[i] = (int *)malloc(m*sizeof(int));
```

显而易见，这种代码是谈不上优雅的，不仅容易出错，而且低效，使用完之后还得用另一个循环来释放内存。

## C语言动态二维数组的优雅使用方式

经过上面的分析，我们知道了要正确的使用C语言的动态二维数组是比较麻烦的。
不过幸好的是，我们有另外的办法来避免上述麻烦。

示例代码如下：

``` c++
int static_array[16*16], *array;

if (m*n < sizeof(static_array))
    array = static_array;
else
    array = (int *)malloc(m*n*sizeof(int));

#define ARRAY(row, col) (*(array + row*m + col))

ARRAY(i, j) = 1;
```
