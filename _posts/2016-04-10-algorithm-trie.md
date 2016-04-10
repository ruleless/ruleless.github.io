---
layout: post
title: "字典树"
description: ""
category: algorithm
tags: [算法, 字典树]
---
{% include JB/setup %}

有一个存放英文单词的文本文件，现在需要知道某些给定的单词是否在该文件中存在，若存在，它又出现了多少次？

这样的问题解法有多种，普通青年直接暴力查找，稍文艺点的用map。
顺序查找的话，每给定一个单词就得遍历整个字符串数组，时间开销实在太大；
如果将所有的单词都存放在一个map中，每次查找的时间复杂度则降为O(log(n))。
不得不说，对于一般的应用场景，map足够满足所有需求。

但对于这个问题，还有更好的方法。有一种数据结构似乎就是为解决此类问题而生的，它就是字典树。
我们知道，英文单词有很多都有相同的前缀，对于相同的前缀，字典树只存储一次，
而map则是有多少个单词就存储多少次，所以在这个问题中，字典树比map省空间；
在字典树中查找字符串的时间复杂度只跟树的深度有关而跟究竟有多少个字符串无关，
而树的深度只跟字符串的长度有关，超过30个拉丁字母的英文单词基本没有，
所以在该问题中查找字符串的时间复杂度只有O(1)，比map省时间。

所以对于这样一个特定的问题，字典树自然是最佳的选择。

对于字典树的空间结构以及如何在字典树中查找字符串，不再坠言，一切净在代码中：

``` c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define MAX_TREE_NODE 128

typedef struct _DictTree
{
	int times;
	struct _DictTree* child[MAX_TREE_NODE];
}DictTree, *DictTreePtr;

static DictTreePtr g_root = NULL;

void insert(const char* str)
{
	int i              = 0;
	int len            = strlen(str);
	DictTreePtr prePtr = NULL;
	DictTreePtr nxtPtr = NULL;

	if (!g_root)
	{
        g_root = (DictTreePtr)malloc(sizeof(DictTree));
        memset(g_root, 0, sizeof(DictTree));
	}
	prePtr = g_root;

	for (i = 0; i < len; ++i)
	{
        nxtPtr = prePtr->child[str[i]];
        if (!nxtPtr)
        {
			nxtPtr = (DictTreePtr)malloc(sizeof(DictTree));
			memset(nxtPtr, 0, sizeof(DictTree));
        }
        prePtr->child[str[i]] = nxtPtr;
        prePtr                = nxtPtr;
        if (i == len -1)
			++(prePtr->times);
	}
}

int search(const char* str)
{
	int i = 0;
	int len = strlen(str);
	DictTreePtr prePtr = NULL;
	DictTreePtr nxtPtr = NULL;

	if (!g_root)
        return 0;
	prePtr = g_root;

	for (i = 0; i < len; ++i)
	{
        nxtPtr = prePtr->child[str[i]];
        if (!nxtPtr)
			return 0;
        prePtr = nxtPtr;
	}

	return prePtr->times;
}
```
