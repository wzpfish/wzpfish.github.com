---
layout: post
title: python virtualenv使用
category: another
---

# python virtualenv

## 介绍

virtualenv是用来产生隔离的python环境的，它不受系统权限限制。

为什么要用virtualenv？

* 如果不同的应用需要不同的lib版本怎么办？全放在`/usr/lib/python2.7/site-packages`显然不科学，因为如果不小心update了某个lib，可能两个都更新了。
* 如果你有个应用写好了想放在那不管它，不去更新了。那么你的lib也不能更新，因为更新了可能该应用就没法用了。
* 如果你没有在全局site-packages安装lib的权限，咋办？

## 安装

	sudo pip install virtualenv

## 常用操作

1. 创建一个新的python环境

		virtualenv ENV
	
	ENV是自己指定的目录，用来放置新的虚拟环境，目录中会产生以下文件夹：
	* ENV/lib/和ENV/include/ 存放新的python库文件，ENV/lib/pythonX.X/site-packages/放新环境的python包
	* ENV/bin存放可执行文件，如pip，easy_install, python等命令。

2. 切换到虚拟环境
	
		source ENV/bin/activate
	
	这时候python环境变量就是虚拟环境的了，所有安装的包以及python操作都在虚拟环境中。	
3. 退出虚拟环境
		
		deactivate
	
## 其他操作

支持一些其他操作，比如创建继承global环境的虚拟环境。

参考[官方文档](https://virtualenv.pypa.io/en/latest/userguide.html)
{% include references.md %}
