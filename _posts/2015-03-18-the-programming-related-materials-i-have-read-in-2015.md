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
资料：iptables防火墙原理详解 | by seanlook | http://segmentfault.com/blog/seanlook/1190000002540601  
评论:对数据包通过一个节点时的处理流程有了全面的了解，里面的图片很不错。再配合着看完网上的《2小时玩转iptables企业版.ppt》，终于有了俯瞰整个数据包走向流程的视野。

时间：02.08 ~ 02.09   
资料：What Every Programmer Should Know about Memory? | by Ulrich Drepper | http://www.akkadia.org/drepper/cpumemory.pdf  
评论：论文篇幅太长，再者涉及内容偏重硬件设计，所以暂时搁置，有时间再读。

时间：03.03  
资料：白话经典算法系列之六 快速排序 快速搞定 | by MoreWindows | http://blog.csdn.net/morewindows/article/details/6684558  
评论：精简地把快速排序的一种思路用精妙的“挖坑待种”的比喻表达了出来，看完并实验后发现这种思路下的快速排序的实现是如此简单！

时间：03.10  
资料：白话经典算法系列之七 堆与堆排序 | by MoreWindows | http://blog.csdn.net/morewindows/article/details/6709644/  
评论：其实作为非科班出身的编程者，在做了很多表层的编程语言层面和公司业务层面的编程工作后，最基础的内功却几乎是空的。数据结构这一块一直是只对编程语言中的常见数据结构有接触，像更底层却很有意思的比如"堆"却几乎没接触过，所幸接触后发现至少堆结构还是很简单的。

时间：03.17  
资料：What every web developer must know about URL encoding | by Stéphane Épardaud | http://blog.lunatech.com/2009/02/03/what-every-web-developer-must-know-about-url-encoding  
评论：这是"编程人员应该知道的xxx"系列中的一篇，主要讲URL-Encoding的种种。其实看的时候更多的是共鸣（相比于其他资料带来的透彻感），因为这样的问题在自己搞开发的时候都已经考虑、验证过了。不过作者讲得更系统、全面一些，看完后至少开阔了眼界（看似乱码实则符合RFC规范的URL，url encode/decode在不同层级的服务上的处理情况如web-server层、编程语言层）

时间：03.17  
资料：What Every Programmer Should know about Time | by EmilMikulic | https://unix4lyfe.org/time/?v=1   
评论：文章挺短的，还趁机看了一下其中提到的"leap seconds"和"Daylight saving"，还顺带回顾了一下两种用来计量时间的装置：石英钟和原子钟的原理。很喜欢文章中提到的一个观点：时间总是应该以Unix Timestamp的形式被保存起来，而不是"Y-m-D H:M:S"这种人类可读的方式，因为后者是表现层的事儿。
