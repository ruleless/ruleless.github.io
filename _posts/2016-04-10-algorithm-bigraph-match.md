---
layout: post
title: "二分图匹配"
description: ""
category: algorithm
tags: []
---
{% include JB/setup %}

二分图匹配是可以转换为网络流求解的，但直接用匈牙利算法求解比转换为网络流求解更高效。

关于二分图有很多公式，比如它与最小顶点覆盖的关系、与最小路径覆盖的关系等等。这里就不进一步讨论了。

``` c++
#define N 1000
int n;//二分图左边节点数
int m;//二分图右边节点数
bool edge[N][N],vis[N];
int link[N];

bool Find(int a)//判断从二分图左边节点a是否可以在右边找到匹配节点
{
    int i;
    for(i=1;i<=m;i++)
    {
        if(edge[a][i]&&!vis[i])
        {
            vis[i]=true;
            if(link[i]==-1||Find(link[i]))
            {
                link[i]=a;
                return true;
            }
        }
    }
    return false;
}

int solve()
{
    int i,k=0;
    memset(link,-1,sizeof(link));
    for(i=1;i<=n;i++)
    {
        memset(vis,0,sizeof(vis));
        if(Find(i))
            k++;
    }
    return k;
}
```

注意到，上边的算法中，右边的每个顶点只能匹配一个左边的顶点，那么如果要求其能匹配多个呢？
这时候就要用到二分图的多重匹配算法了。算法思想与上述并无二致。

``` c++
#define N 1000
int n,m,link[N][N];//link[i][j]表示右边的第i个节点在左边的第j个匹配节点，
//link[i][0]表示i节点当前已匹配数目
bool edge[N][N],vis[N];

bool Find(int a,int lim)//lim表示右边的每个节点的最多匹配数目
{
    int i,j;
    for(i=1;i<=m;i++)
    {
        if(edge[a][i]&&!vis[i])
        {
            vis[i]=true;
            if(link[i][0]<lim)
            {
                link[i][++link[i][0]]=a;
                return true;
            }
            else
            {
                for(j=1;j<=link[i][0];j++)
                {
                    if(Find(link[i][j],lim))
                    {
                        link[i][++link[i][0]]=a;
                        return true;
                    }
                }
            }
        }
    }
    return false;
}

int solve()
{
    int i,k=0,lim;
    memset(link,0,sizeof(link));
    for(i=1;i<=n;i++)
    {
        memset(vis,0,sizeof(vis));
        if(Find(i,lim))
            k++;
    }
    return k;
}
```
