---
layout: post
title: ������֮Push Relabel�㷨
category: another
---


׼����ʼ����ʰ���㷨����

������㷨��Ҫ�������࣬һ��������**����·**����һ����**Push Relabel**���뷨��

���쿴�㷨����ѧϰ�����������㷨�Ļ���˼�룬��ƪ��Ҫ������ʵ��Push Relabel�㷨ʱ��һЩ�ջ�

������Push��������һ���ڵ� u ��������� (u,v) �ǲд��ʱ����u����ʹ��Push������

```
void Push(int u,int v){
	int minF = min(e[u],cf[u][v]);  //u���������(u,v)�д�������Сֵ��
	cf[u][v] -= minF;
	cf[v][u] += minF;
	e[u] -= minF;
	e[v] += minF;   
}
```

Relabel��������һ���ڵ�u��������Ҷ������еı� (u,v) ����u.h <= v.h����u����ʹ��Rebel������**Note:���������У���һ���ڵ��������Ҫô���Զ���Push��Ҫô���Զ���Relabel��**

```
void Relabel(int u){
	int minH = INF;  //����Ҫ����һ���ܴ������������Ϊ��Relabel����ʱ���������ĳ���ڵ��hֵ��Դ���hֵs.h��ö�  ��ʱ���⿨�˰���
	for(int i=0;i<=t;++i){
		if(cf[u][i]>0) minH = min(minH,h[i]);
	}
	h[u] = minH + 1;
}
```

{% include references.md %}