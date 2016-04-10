---
layout: post
title: "SPFA(Shortest path faster algorithm)"
description: ""
category: algorithm
tags: [算法, spfa]
---
{% include JB/setup %}

## 算法简介

SPFA算法是求单源最短路径的一种算法，它还有一个重要的功能是判负环（在差分约束系统中会得以体现）。
SPFA算法在Bellman-ford算法的基础上加了一个队列优化，减少了冗余的松弛操作，是一种高效的最短路算法。

## 名词阐释

  * N：顶点数
  * edge[i][j]：顶点 i 到顶点 j 的距离
  * s：源点
  * dist：最短路估值数组
  * Que：队列

## 算法流程

我们可用如下伪代码表述 SPFA 的算法流程：

``` c++
初始化 dist[s] 为0，其他为 Inf，将 s 插入 Que 队尾;

while (Que 非空)
{
	u = Que.pop();

	for (v connect to u)
	{
		if (dist[u]+edge[u][v] < dist[v])
		{
			dist[v] = dist[u] + edge[u][v];
			if (v is not in the Que)
			{
				Que.push(v);
			}
		}
	}
}
```

## 示例代码

我们以 [POJ 3259](http://poj.org/problem?id=3259) 为例给出 SPFA 算法的一个示例：

``` c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define Inf 9999999
#define N 505

struct Edge
{
	int to, w, next;
} s_Edge[N*N];
int edgecount = 0;
int vertex[N], n;

int Que[N*N];
int head, tail;

bool spfa(int s, int dist[])
{
	head = tail = 0;
	for (int u = 1; u <= n; ++u)
		dist[u] = Inf;
	dist[s] = 0;

	bool bInQue[N];
	int nPushSum[N];
	memset(bInQue, 0, sizeof(bInQue));
	memset(nPushSum, 0, sizeof(nPushSum));
	Que[tail++] = s;
	bInQue[s] = true;
	nPushSum[s]++;
	while (head < tail)
	{
		int u = Que[head++];
		bInQue[u] = false;

		for (int e = vertex[u]; e != -1; e = s_Edge[e].next)
		{
			int v = s_Edge[e].to;
			if (dist[u] + s_Edge[e].w < dist[v])
			{
				dist[v] = dist[u] + s_Edge[e].w;
				if (!bInQue[v])
				{
					Que[tail++] = v;
					bInQue[v] = true;
					if (++nPushSum[v] >= n)
						return false;
				}
			}
		}
	}
	return true;
}

int Graph[N][N], gNodeCnt;
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

		// 构图
		n = gNodeCnt;
		memset(vertex, -1, sizeof(vertex));
		edgecount = 0;
		for (int u = 1; u <= n; ++u)
		{
			for (int v = 1; v <= n; ++v)
			{
				if (Graph[u][v] != 0 && Graph[u][v] != Inf)
				{
					s_Edge[edgecount].to = v;
					s_Edge[edgecount].w = Graph[u][v];
					s_Edge[edgecount].next = vertex[u];
					vertex[u] = edgecount++;
				}
			}
		}

		int dist[N];
		if (spfa(1, dist))
			printf("NO\n");
		else
			printf("YES\n");
	}
	return 0;
}
```
