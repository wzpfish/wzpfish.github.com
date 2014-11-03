---
layout: post
title: 搭博客的一些感悟续
category: default
---

早上来到实验室又想着把博客优化一下..Down下来的模版里的图片都是在线的，总是加载失败，果断放到本地了。。

接触了DNSPOD，又快又强大的DNS解析居然免费，一想到tk自己的dns服务器这么弱，就把它托管给DNSPOD了。友情链接：<https://www.dnspod.cn>

在github上使用自定义域名，只要添加一个CNAME文件就可以了，第一次生效大概要10-15分钟吧。接着，如果用的是Apex domain,就在DNSPOD上添加A记录，用的是Subdomain，就添加C记录，把记录值指向github空间的IP地址就可以了。

后来觉得博客还是有些简陋，就用Disqus加了个评论插件，因为github上只能是静态网页，Disqus是把评论托管在它的网站上，自己可以登录上去管理这些评论，挺方便的。友情链接：<https://disqus.com/>

好了，博客界面大概就这样吧. 接下来就保持热情更新博文，说不定以后回头看看会有一些收获呢。
{% include references.md %}
	
