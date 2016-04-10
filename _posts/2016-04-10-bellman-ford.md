---
layout: post
title: "Bellman-Ford算法求最短路"
description: ""
category: algorithm
tags: [算法, 最短路]
---
{% include JB/setup %}

Bellman-ford算法求单源最短路径算法的基本思路是：

> 进行不停地松弛，每次松弛把每条边都更新一下，若 n-1  次松弛后还能更新，
则说明图中有负环，无法得出结果，否则就成功完成。

## 名词阐释

  * G：图
  * edge：一条边，edge.u 表示该边起点，edge.v 表示终点，edge.w 表示边的权值
  * N：顶点数
  * edge[i][j]：顶点 i 到顶点 j 的距离
  * s：源点
  * dist：最短路径估值数组

## 算法流程

``` c++
初始化 dist[s] 为0，其他为 Inf;

for (i = 1, n-1)
{
	for (every edge in G)
	{
		if (dist[edge.u] + edge.w < dist[edge.v])
		{
			dist[edge.v] = dist[edge.u] + edge.w;
		}
	}
	若此次没有更新操作，则表示成功找到所有点到 s 的最短路径;
}

若任意边仍可进行松弛更新操作，则表示存在负权回路;
```

## 示例代码

我们以 [POJ 3259](http://poj.org/problem?id=3259) 为例给出 Bellman-Ford 算法的一个示例：

``` c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define Inf 9999999
#define N 505
int Graph[N][N], gNodeCnt;

bool bellman_ford(int edge[N][N], int n, int s, int dist[])
{
	for (int u=1; u <= n; ++u)
		dist[u] = Inf;
	dist[s] = 0;

	for (int k = 1; k <= n-1; ++k)
	{
		bool flag = true;
		for (int u = 1; u <= n; ++u)
		{
			for (int v = 1; v <= n; ++v)
			{
				if (edge[u][v] < Inf && dist[u]+edge[u][v] < dist[v])
				{
					dist[v] = dist[u]+edge[u][v];
					flag = false;
				}
			}
		}

		if (flag)
			return true;
	}
	for (int u = 1; u <= n; ++u)
	{
		for (int v = 1; v <= n; ++v)
		{
			if (edge[u][v] < Inf && dist[u]+edge[u][v] < dist[v])
				return false;
		}
	}
	return true;
}

int main()
{
	// freopen("in.txt", "r", stdin);

	int F;
	scanf("%d", &F);
	while (F--)
	{
		int M, W;
		scanf("%d%d%d", &gNodeCnt, &M, &W);

		for (int i = 1; i <= gNodeCnt; ++i)
		{
			for (int j = i; j <= gNodeCnt; ++j)
			{
				Graph[i][j] = Graph[j][i] = (i == j ? 0 : Inf);
			}
		}
		while (M--)
		{
			int s, t, w;
			scanf("%d%d%d", &s, &t, &w);
			if (w < Graph[s][t])
			{
				Graph[t][s] = w;
				Graph[s][t] = w;
			}
		}
		while (W--)
		{
			int s, t, w;
			scanf("%d%d%d", &s, &t, &w);
			w = -w;
			if (w < Graph[s][t])
				Graph[s][t] = w;
		}

		int dist[N];
		if (bellman_ford(Graph, gNodeCnt, 1, dist))
			printf("NO\n");
		else
			printf("YES\n");
	}
	return 0;
}
```
