---
layout: post
title: URL中的Hostname VS RequestHeader中的Host字段
---

## 概述
通常情况下访问URL的时候，我们直接使用一个字段标明这个URL所属的HOST信息

但是如果你的URL中的HostName是IPV4地址，同时在请求的Header中设置了"Host"字段，那么这个资源(URI)的寻找依据会是什么呢？

## 准备步骤：
1. 把`i-dev.beibei.com`的host指向`192.168.33.10` (本机eth1的网卡IP地址)
2. 访问`http://127.0.0.1/account/coupon.html`，同时设置RequetHeader的Host字段为 `i-dev.beibei.com`
3. 开始访问: `curl -v --head --header "host: i-dev.beibei.com" `
 (-v指明verbose详细信息；--head指明打印RequestHeader；--header用于设置RequestHeader的字段)

![](http://upload-images.jianshu.io/upload_images/4111-60fda1fdbe323bbd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过curl结果可以看到，寻找这个资源时，HTTP1.1 优先通过url中的ipv4地址和80端口直连web-server，RequestHeader中的Host绑定的ip：192.168.33.10并未出现在请求过程中

## 应用场景
#### 场景1
现有：一个HTTP的调用方(例如RocketMQ的proxy)，同时有一个HTTP的提供方（server，例如www.beibei.com）。

如何做到将这两个程序部署在同一机器上，而且HTTP调用方只会访问本机的HTTP提供方的服务？

方法1：改动/etc/hosts文件，将host指向本机。方案点评：如果部署100台机器，就需要改动100次hosts文件。

方法2：访问http://127.0.0.1/，同时在HTTP Request的Header中设置host为`www.beibei.com`
方案点评：只需部署代码，无需改动机器配置

#### 场景2
另一个应用场景：在本地开发时，需要触发某个URL被访问时，可以这样：
`curl --header "host: 
 www.beibei.com" http://127.0.0.1/preview/detail.html`