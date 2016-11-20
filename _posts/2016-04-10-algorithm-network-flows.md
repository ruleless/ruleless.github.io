---
layout: post
title: "网络流"
description: "网络流"
category: algorithm
tags: []
---
{% include JB/setup %}

如果只从网络流的定义来看，它能求解的问题似乎很少，但如果考虑到转换的思想，
则很多看起来与网络流丝毫不沾边的问题也能用网络流来解，
这也是为什么网络流在ACM比赛中被列为较难题的原因所在。

我这里不打算讨论怎么对一个问题建立网络流模型，说句实在话，我也无从讨论这个。
如果将适于用网络流解决的问题看成是一个集合，那么这个集合是趋于无穷的，
如果按照某种规则对它们进行分类组成另外一个问题分类集，那么这个问题分类集也是趋于无穷的。
所以，唯一的办法是训练自己对这类问题的敏锐性，然后具体问题具体分析。

Edmonds Karp求最大流的算法：

``` c++
#define N 1000
#define Inf 9999999

int G[N][N],c[N][N],f[N][N],pre[N],que[N],n,vis[N];

int ek(int s,int t)
{
	int i,j,k,head,tail,flow=0;
	memset(f,0,sizeof(f));
	while(true)
	{
		head=tail=0;
		memset(vis,0,sizeof(vis));
		que[tail++]=s;
		vis[s]=true;
		while(head<tail)
		{
			k=que[head++];
			if(k==t)
				break;
			for(i=1;i<=n;i++)
			{
				if(c[k][i]>0&&!vis[i])
				{
					vis[i]=true;
					pre[i]=k;
					que[tail++]=i;
				}
			}
		}
		if(k!=t)
			break;
		int cc=Inf;
		j=t;
		i=pre[j];
		while(j!=s)
		{
			if(c[i][j]<cc)
				cc=c[i][j];
			j=i;
			i=pre[j];
		}
		flow+=cc;
		j=t;
		i=pre[j];
		while(j!=s)
		{
			f[i][j]+=cc;
			f[j][i]=-f[i][j];
			c[i][j]=G[i][j]-f[i][j];
			c[j][i]=G[j][i]-f[j][i];
			j=i;
			i=pre[j];
		}
	}
	return flow;
}
```

dinic求最大流，图用邻接矩阵表示：

``` c++
#define N 1000
#define Min(a,b) a<b?a:b
#define Inf 9999999
int G[N][N],c[N][N],n,s,t;
int level[N],que[N];

bool makelevel()
{
	int i,j,k,head=0,tail=0;
	memset(level,-1,sizeof(level));
	level[s]=0;
	que[tail++]=s;
	while(head<tail)
	{
		k=que[head++];
		for(i=1;i<=n;i++)
			if(c[k][i]>0&&level[i]==-1)
			{
				level[i]=level[k]+1;
				que[tail++]=i;
			}
	}
	return level[t]!=-1;
}

int findpath(int u,int alpha)
{
	if(u==t)
		return alpha;
	int i,j,k,w;
	w=0;
	for(i=1;i<=n;i++)
	{
		if(c[u][i]>0&&level[i]==level[u]+1)
		{
			if(k=findpath(i,Min(c[u][i],alpha-w)))
			{
				c[u][i]-=k;
				c[i][u]+=k;
				w+=k;
			}
		}
	}
	return w;
}

int dinic()
{
	int i,j,k=0;
	while(makelevel())
		while(i=findpath(s,Inf))
			k+=i;
	return k;
}
```

dinic求最大流，图用静态邻接表表示：

``` c++
#define Min(a,b) a<b?a:b
#define N 10000
#define Inf 9999999
struct Edge
{
	int to,cap,opt,next;
}e[100000];
int ec,pp[N],n,s,t;
int level[N],que[N];

bool makelevel()
{
	int i,j,k,head=0,tail=0;
	memset(level,-1,sizeof(level));
	level[s]=0;
	que[tail++]=s;
	while(head<tail)
	{
		k=que[head++];
		for(i=pp[k];i!=-1;i=e[i].next)
			if(e[i].cap>0&&level[e[i].to]==-1)
			{
				level[e[i].to]=level[k]+1;
				que[tail++]=e[i].to;
			}
	}
	return level[t]!=-1;
}

int findpath(int u,int alpha)
{
	if(u==t)
		return alpha;
	int i,j,k,w=0;
	for(i=pp[u];i!=-1;i=e[i].next)
		if(e[i].cap>0&&level[e[i].to]==level[u]+1)
			if(k=findpath(e[i].to,Min(e[i].cap,alpha-w)))
			{
				e[i].cap-=k;
				e[e[i].opt].cap+=k;
				w+=k;//多路增广
			}
	return w;
}

int dinic()
{
	int i,j,k=0;
	while(makelevel())
		while(i=findpath(s,Inf))
			k+=i;
	return k;
}
```

sap算法求最大流，图用邻接矩阵表示：

``` c++
#define N 10000
#define Inf 99999999
struct Edge
{
	int to,cap,opt,next;
}e[100000];
int ec,pp[10000],n;//pp[i]用于保存第i个节点的边,n为图的节点数目
int que[N],dist[N],gap[2*N],pre[N],cur[N];

void bfs(int t)
{
	int i,j,k,head=0,tail=0;
	memset(dist,0,sizeof(dist));
	memset(gap,0,sizeof(gap));
	gap[0]=1;
	que[tail++]=t;
	while(head<tail)
	{
		k=que[head++];
		for(i=pp[k];i!=-1;i=e[i].next)
		{
			int v=e[i].to;
			if(!dist[v]&&v!=t)
			{
				dist[v]=dist[k]+1;
				gap[dist[v]]++;
				que[tail++]=v;
			}
		}
	}
}

int sap(int s,int t)
{
	int i,j,k,flow=0,u=s,neck;
	memcpy(cur,pp,sizeof(pp));
	bfs(t);
	while(dist[s]<n)
	{
		if(u==t)
		{
			int cc=Inf;
			for(i=s;i!=t;i=e[cur[i]].to)
				if(e[cur[i]].cap<cc)
				{
					neck=i;
					cc=e[cur[i]].cap;
				}
			flow+=cc;
			for(i=s;i!=t;i=e[cur[i]].to)
			{
				e[cur[i]].cap-=cc;
				e[e[cur[i]].opt].cap+=cc;
			}
			u=neck;
		}
		for(i=cur[u];i!=-1;i=e[i].next)
			if(e[i].cap>0&&dist[u]==dist[e[i].to]+1)
				break;
		if(i!=-1)
		{
			cur[u]=i;
			pre[e[i].to]=u;
			u=e[i].to;
		}
		else
		{
			if(--gap[dist[u]]==0)break;
			cur[u]=pp[u];
			int tmp=n-1;
			for(i=pp[u];i!=-1;i=e[i].next)
				if(e[i].cap>0&&dist[e[i].to]<tmp)
					tmp=dist[e[i].to];
			dist[u]=tmp+1;
			gap[dist[u]]++;
			if(u!=s)u=pre[u];
		}
	}
	return flow;
}
```

sap算法求最大流，图用静态邻接表表示：

``` c++
#define N 10000
#define Inf 99999999
struct Edge
{
	int to,cap,opt,next;
}e[100000];
int ec,pp[10000],n;//pp[i]用于保存第i个节点的边,n为图的节点数目
int que[N],dist[N],gap[2*N],pre[N],cur[N];

void bfs(int t)
{
	int i,j,k,head=0,tail=0;
	memset(dist,0,sizeof(dist));
	memset(gap,0,sizeof(gap));
	gap[0]=1;
	que[tail++]=t;
	while(head<tail)
	{
		k=que[head++];
		for(i=pp[k];i!=-1;i=e[i].next)
		{
			int v=e[i].to;
			if(!dist[v]&&v!=t)
			{
				dist[v]=dist[k]+1;
				gap[dist[v]]++;
				que[tail++]=v;
			}
		}
	}
}

int sap(int s,int t)
{
	int i,j,k,flow=0,u=s,neck;
	memcpy(cur,pp,sizeof(pp));
	bfs(t);
	while(dist[s]<n)
	{
		if(u==t)
		{
			int cc=Inf;
			for(i=s;i!=t;i=e[cur[i]].to)
				if(e[cur[i]].cap<cc)
				{
					neck=i;
					cc=e[cur[i]].cap;
				}
			flow+=cc;
			for(i=s;i!=t;i=e[cur[i]].to)
			{
				e[cur[i]].cap-=cc;
				e[e[cur[i]].opt].cap+=cc;
			}
			u=neck;
		}
		for(i=cur[u];i!=-1;i=e[i].next)
			if(e[i].cap>0&&dist[u]==dist[e[i].to]+1)
				break;
		if(i!=-1)
		{
			cur[u]=i;
			pre[e[i].to]=u;
			u=e[i].to;
		}
		else
		{
			if(--gap[dist[u]]==0)break;
			cur[u]=pp[u];
			int tmp=n-1;
			for(i=pp[u];i!=-1;i=e[i].next)
				if(e[i].cap>0&&dist[e[i].to]<tmp)
					tmp=dist[e[i].to];
			dist[u]=tmp+1;
			gap[dist[u]]++;
			if(u!=s)u=pre[u];
		}
	}
	return flow;
}
```
