---
layout: post
title: "双连通分量"
description: ""
category: algorithm
tags: [算法, 双连通分量]
---
{% include JB/setup %}

在无向连通图中，如果删除该图的任何一个结点都不能改变该图的连通性，则称该图是双连通的。
双连通无向图一定是连通的，而连通的无向图则不一定是双连通的。
对于一个连通的无向图也有双连通分量的概念，定义自然不言而喻。

同样，我们也可以利用tarjan算法求双连通分量。

``` c++
#define N 10000
struct Edge
{
    int to,next;
}e[100000];
int ec,pp[N],n;
int dfn[N],low[N],stafrom[100000],stato[100000],top,index,cnt;
set<int>Set[N];

void tarjan(int u,int p)
{
    int i,j,k,v,x,y;
    dfn[u]=low[u]=++index;
    for(i=pp[u];i!=-1;i=e[i].next)
    {
        v=e[i].to;
        if(p!=v&&dfn[v]<dfn[u])
        {
            stafrom[++top]=u;
            stato[top]=v;
            if(!dfn[v])
            {
                tarjan(v,u);
                if(low[v]<low[u])
                    low[u]=low[v];
                if(dfn[u]<=low[v])
                {
                    do{
                        x=stafrom[top];
                        y=stato[top--];
                        Set[cnt].insert(x);
                        Set[cnt].insert(y);
                    }while( !(x==u&&y==v) );
                    cnt++;
                }
            }
            else if(dfn[v]<low[u])
                low[u]=dfn[v];
        }
    }
}

void solve()
{
    int i,j,k;
    index=top=cnt=0;
    memset(dfn,0,sizeof(dfn));
    for(i=1;i<=n;i++)
        if(!dfn[i])
            tarjan(i,-1);
}
```

代码中，cnt表示双连通分量的个数，Set数组用来存储双连通分量。
