---
title: zaver学习记录
date: 2020-09-27 23:40
tags: [开源,网络]
categories: 编程
toc: true
---

<img src="https://s1.ax1x.com/2020/09/27/0AY0gS.jpg" alt="epoll" style="zoom:40%;" />

zaver是一个高性能的http服务器，使用了epoll作为事件驱动机制，同时运用了一些巧妙的数据结构，架构方面参考了nginx的实现。记录分享一下关于zaver的学习过程。

带详细中文注释的代码放在了码云上。链接：[zaver](https://gitee.com/zhanghh0624/zaver)

<!--more-->