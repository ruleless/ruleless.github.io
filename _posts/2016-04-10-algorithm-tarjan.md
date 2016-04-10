---
layout: post
title: "强连通分量"
description: ""
category: algorithm
tags: [算法, 强连通分量]
---
{% include JB/setup %}

对于有向图的一个顶点集，如果从这个顶点集的任何一点出发都可以到达该顶点集的其余各个顶点，
那么该顶点集称为该有向图的一个强连通分量。有向连通图的全部顶点组成一个强连通分量。

我们可以利用tarjan算法求强连通分量：

``` c++
#define N 1000
struct Edge
{
	int to,next;
}e[100000];
int ec,pp[N],n;
int dfn[N],low[N],stack[N],index,top,cnt;
bool instack[N];
set<int>Set[N];

void tarjan(int u)
{
	int i,j,k,v;
	dfn[u]=low[u]=++index;
	stack[++top]=u;
	instack[u]=true;
	for(i=pp[u];i!=-1;i=e[i].next)
	{
		v=e[i].to;
		if(!dfn[v])
		{
			tarjan(v);
			if(low[v]<low[u])
				low[u]=low[v];
		}
		else if(instack[v]&&dfn[v]<low[u])
			low[u]=dfn[v];
	}
	if(dfn[u]==low[u])
	{
		do{
			j=stack[top--];
			instack[j]=false;
			Set[cnt].insert(j);
		}while(j!=u);
		cnt++;
	}
}

void solve()
{
	int i,j,k;
	top=index=cnt=0;
	memset(instack,0,sizeof(instack));
	memset(dfn,0,sizeof(dfn));
	for(i=1;i<=n;i++)
		if(!dfn[i])
			tarjan(i);
}
```

代码中cnt保存的是强连通分量的个数，Set数组用来保存每一个强连通分量。
