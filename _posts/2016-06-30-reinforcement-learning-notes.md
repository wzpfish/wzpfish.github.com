---
layout: post
title: python virtualenv使用
category: another
---

# Notes for Reinforcement learning

上个月跟着silver的视频学了一下RL，最近看的时候感觉全忘了..于是重新整理了一下笔记，以便忘了的时候翻翻..

silver的课程地址: http://www0.cs.ucl.ac.uk/staff/d.silver/web/Teaching.html.

## introduction
第一讲介绍了RL的一些基本概念，比如Reward，Agent，Environment，state，action，policy，value function和model。
概念比较简单，就不写了。

## MRP and MDP
这讲介绍了MRP和MDP的基本定义.主要需要掌握：

* MRP
    1. Return
	2. state-value function
	3. bellman equation
* MDP
    1. policy
	2. state-value function and action-value function
	3. bellman equation
	4. optimal state-value function and optimal action-value function
	5. bellman optimality equation
	
详见手写笔记..
![MRP](/img/RL/MRP.jpg)
![MRP](/img/RL/MDP.jpg)
![MRP](/img/RL/MDP2.jpg)

## Planning by DP
这讲主要介绍用DP算法来解决MDP的planning问题，包括prediction和control： prediction是给定MDP的model和policy，求出value function； 而control是给定MDP的model，求出最优的policy.

DP的解决方案（synchronous和asynchronous）：

* For Prediction:
	1. iterative policy evaluation
* For Control:
	1. Policy Iteration
	2. Generalised Policy Iteration
	3. Value iteration

最后总结了DP的优点特点及不足。

详见手写笔记..
![MRP](/img/RL/DP.jpg)
![MRP](/img/RL/DP2.jpg)
![MRP](/img/RL/DP3.jpg)

{% include references.md %}
