---
title: github.io无法打开
date: 2022-03-16 12:53:06
tags:
- github.io
categories:
- Blog
---

##### 1.github.io无法打开

今天发现自己的博客hezd.github.io无法打开，打开VPN代理必须设置全局代理才能打开，这种体验十分槽糕，为了解决这个问题，查阅了很多资料最终解决。

##### 2.解决办法

控制面板>网络连接和Internet>更改适配器设置,然后找到自己连接无线网络，右键属性>Internet协议版本4,双击打开设置DNS服务器

有两种方法

1.只设置首选DNS服务器为114.114.114.114

2.设置首选DNS服务器为223.5.5.5，备用DNS服务器223.6.6.6

我是用第二种方法测试成功打开我的博客https://hezd.github.io/

##### 参考

https://segmentfault.com/a/1190000039848385
