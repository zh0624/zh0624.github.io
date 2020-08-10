---
title: 多路I/O复用之epoll
date: 2020-08-10 23:45:00
tags: [同步,操作系统]
categories: 编程
toc: true
---

<img src="https://s1.ax1x.com/2020/08/11/aq5Z0f.jpg" alt="epoll" style="zoom:40%;" />

I/O多路复用是通过一种机制，让进程可以同时监听多个文件描述符，当某个文件描述符符合某种状态时，能够通知到相应的进程采取对应的操作。多路复用机制发展至现在，常用的有select、poll、epoll三种方式，它们的发展历程是一个不断优化的过程。本文首先简要介绍多路复用机制的概念，再简单描述下它的发展例程，会着重介绍epoll的机制及部分实现。文中内容提炼了不少前人梳理的结果，并加以自己的理解进行阐述。

<!--more-->

参考文章：

[1] [深入理解Epoll](https://zhuanlan.zhihu.com/p/93609693)

[2] [IO多路复用的三种机制Select，Poll，Epoll](https://www.jianshu.com/p/397449cadc9a)

[3] [IO多路复用机制详解](https://www.cnblogs.com/yanguhung/p/10145755.html)

[4] [如果这篇文章说不清epoll的本质，那就过来掐死我吧！](https://zhuanlan.zhihu.com/p/63179839)