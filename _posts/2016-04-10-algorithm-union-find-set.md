---
layout: post
title: "并查集"
description: ""
category: algorithm
tags: []
---
{% include JB/setup %}

并查集是一种树型的数据结构，用于处理不相交集合的合并及查询问题，在使用中常常以森林来表示。

## 结构说明

我们可以用一个简单的数组表示一个森林，数组可定义为 `parent[N]`，`parent[v]` 表示节点 `v` 的父节点。
当 `v` 没有父节点时（即 `v` 为根节点）：`parent[v]<0`，`|parent[v]|` 表示节点 `v` 拥有多少个子节点和子孙节点。

## 查询操作

``` c++
int Find(int a)
{
	if (parent[a] < 0)
		return a;

	int root = Find(parent[a]);
	parent[a] = root; // 路径压缩
	return root;
}
```

## 合并操作

``` c++
void Union(int a, int b)
{
	int rootA = Find(a);
	int rootB = Find(b);
	if (rootA == rootB)
		return;

	if (parent[rootA] < parent[rootB])
	{
		parent[rootA] += parent[rootB];
		parent[rootB] = rootA;
	}
	else
	{
		parent[rootB] += parent[rootA];
		parent[rootA] = rootB;
	}
}
```
