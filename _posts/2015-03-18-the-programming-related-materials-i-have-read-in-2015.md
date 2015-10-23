---
layout: post
title: "我在2015年读过的编程相关的资料"
description: ""
tags: [资料]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: true
share: true
---

平时会读一些技术博客什么的，但都是零零星星不成体统的。所以在此新建一篇博客来专门记录这些自己读过的编程相关的资料。

<!--more-->

时间：02.07  
资料：《iptables防火墙原理详解》 | by seanlook | http://segmentfault.com/blog/seanlook/1190000002540601  
评论:对数据包通过一个节点时的处理流程有了全面的了解，里面的图片很不错。再配合着看完网上的《2小时玩转iptables企业版.ppt》，终于有了俯瞰整个数据包走向流程的视野。

时间：02.08 ~ 02.09   
资料：《What Every Programmer Should Know about Memory?》 | by Ulrich Drepper | http://www.akkadia.org/drepper/cpumemory.pdf  
评论：论文篇幅太长，再者涉及内容偏重硬件设计，所以暂时搁置，有时间再读。

时间：03.03  
资料：《白话经典算法系列之六 快速排序 快速搞定》 | by MoreWindows | http://blog.csdn.net/morewindows/article/details/6684558  
评论：精简地把快速排序的一种思路用精妙的“挖坑待种”的比喻表达了出来，看完并实验后发现这种思路下的快速排序的实现是如此简单！

时间：03.10  
资料：《白话经典算法系列之七 堆与堆排序》 | by MoreWindows | http://blog.csdn.net/morewindows/article/details/6709644/  
评论：其实作为非科班出身的编程者，在做了很多表层的编程语言层面和公司业务层面的编程工作后，最基础的内功却几乎是空的。数据结构这一块一直是只对编程语言中的常见数据结构有接触，像更底层却很有意思的比如"堆"却几乎没接触过，所幸接触后发现至少堆结构还是很简单的。

时间：03.17  
资料：《What every web developer must know about URL encoding》 | by Stéphane Épardaud | http://blog.lunatech.com/2009/02/03/what-every-web-developer-must-know-about-url-encoding  
评论：这是"编程人员应该知道的xxx"系列中的一篇，主要讲URL-Encoding的种种。其实看的时候更多的是共鸣（相比于其他资料带来的透彻感），因为这样的问题在自己搞开发的时候都已经考虑、验证过了。不过作者讲得更系统、全面一些，看完后至少开阔了眼界（看似乱码实则符合RFC规范的URL，url encode/decode在不同层级的服务上的处理情况如web-server层、编程语言层）

时间：03.17  
资料：《What Every Programmer Should know about Time》 | by EmilMikulic | https://unix4lyfe.org/time/?v=1   
评论：文章挺短的，还趁机看了一下其中提到的"leap seconds"和"Daylight saving"，还顺带回顾了一下两种用来计量时间的装置：石英钟和原子钟的原理。很喜欢文章中提到的一个观点：时间总是应该以Unix Timestamp的形式被保存起来，而不是"Y-m-D H:M:S"这种人类可读的方式，因为后者是表现层的事儿。

时间：03.27  
资料：《An Introduction To DOM Events》| by Wilson Page | http://www.smashingmagazine.com/2013/11/12/an-introduction-to-dom-events/  
评论：最大收获是终于对着w3c的[这张图](http://www.w3.org/TR/DOM-Level-3-Events/eventflow.svg)把一个DOM Event的生命周期看懂了（之前尝试看了一次，模棱两可）。其次是琐碎的事了：了解了`func.bind()`方法

时间：06.12  
资料：《Inside NGINX: How We Designed for Performance & Scale》| by OWEN GARRETT | http://nginx.com/blog/inside-nginx-how-we-designed-for-performance-scale/    
评论：很多架构解读型的文章，最生动和令人难忘的，大概都属随文展示出来的架构图了。这篇文章里列出了好几张非常不错的架构、流程图。它在拿自己的non-blocking设计和其他相对blocking的设计作对比时，举了个非常形象的例子：Web服务器和Web客户端的通信就像下棋一样，只不过Web服务器通常是一对多负责和很多客户端下棋。服务器可以为每个客户端分配一个棋手（blocking，每个线程/进程负责一个客户端），也可以只用一个棋手来同时跟若干个客户端轮询着下棋（non-blocking，现实中有真人1 vs 360同时进行的棋局哦http://t.cn/R2joHP0 ）。

时间：06.25    
资料：《Thread Pools in NGINX Boost Performance 9x!》 | by VALENTIN BARTENEV | http://nginx.com/blog/thread-pools-boost-performance-9x/    
评论：整篇文章其实只是一个结果报告，Event-Driven-Handling，但是handle某些event时操作确实又是blocking的，怎么办？引入线程池。线程池一直都是用的各类语言自带的(java, python)，具体实现并不是很清楚，计划有时间了看一些线程池的实现的资料。

时间：10.23  
资料：《Visual Representation of SQL Joins》 | by C.L. Moffatt | http://www.codeproject.com/Articles/33052/Visual-Representation-of-SQL-Joins  
评论：关系型数据库Join表的方式有哪些，每种join的使用结果如何？在对Mysql有了一定的业务层的使用经验后，再看这篇文章，觉得人家写的真是简洁而全面。
