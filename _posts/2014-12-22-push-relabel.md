---
layout: post
title: 网络流之Push Relabel算法
category: another
---


准备开始重新拾起算法。。

最大流算法主要有两大类，一类是利用**增广路**，另一类是**Push Relabel**的想法。

今天看算法导论学习了这两大类算法的基本思想，这篇主要讲我在实现Push Relabel算法时的一些收获。

首先是Push操作，当一个节点 u 溢出，并且 (u,v) 是残存边时，则u可以使用Push操作。


    void Push(int u,int v){
	    int minF = min(e[u],cf[u][v]);  //u的溢出量和(u,v)残存量的    最小值。
	    cf[u][v] -= minF;
	    cf[v][u] += minF;
	    e[u] -= minF;
	    e[v] += minF;   
    }

Relabel操作，当一个节点u溢出，并且对于所有的边 (u,v) 都有u.h <= v.h，则u可以使用Rebel操作。**Note:在流网络中，若一个节点溢出，那要么可以对它Push，要么可以对它Relabel。**

```
void Relabel(int u){
	int minH = INF;  //这里要设置一个很大的数！！！因为在Relabel操作时，可能最后某个节点的h值比源点的h值s.h大好多  当时做题卡了半天
	for(int i=0;i<=t;++i){
		if(cf[u][i]>0) minH = min(minH,h[i]);
	}
	h[u] = minH + 1;
}
```

{% include references.md %}