---
layout: post
title: "写在离职季，记录我的第一份编程工作"
comments: true
tags: [Tech, 总结]
---

## 一只编程菜鸟，只为找一份编程的工作
13年7月份毕业，听从家里人的安排，去了本省的一家国企工作，直接被派去到东北的项目基地实习。因为会和技术老外讲英语，还和第三方外包人员一起合作调试好了他们承接的电气化自动排水项目的程序，基地领导青睐有加，想把我留在那里。我考虑了一下拒绝了。等工作到12月份的时候，基本已经确定，这份工作不是我想要的。遂瞒着家里人和领导请了一个月的病假，从东北坐了33个小时的火车来到江南，杭州。当时已是冬季，但下火车后仍然觉得“春风拂面”，比起东北让人冻得骨头疼的冷，杭州的天气温和到简直像是春天。

<!--more-->

投靠到朋友那里，开始了投简历、找工作的经历。因为大三才开始自学编程，基础不牢，项目经验少得可怜。找了快一个月的时间，却只有零星的几次线上交流、笔试的机会，offer始终无果。眼看着一个月的假就要耗完，马上就要春节了，工作依然没有着落，心里开始灰心。直到在准备离开杭州的前两天，才终于有了一次面试的机会，这就是目前的这份工作的起点。这是一家小公司，做海外代购的，同时也在做商品聚合平台。聊了一个多小时，人家觉得还不错，遂口头答应了offer。于是终于在离开杭州的那天，收到了这家公司的正式offer。这个offer就像一根救命稻草一般，没有经历过之前的绝望，就根本体会不到这个offer带来的激动和心安：如果找不到工作，就只能回原来的单位上班了，而当初是做了死也要死外边的打算的。

因为这是收到的唯一一个offer，所以在回去后的第三天就接受了这份工作。所幸可以安安心心地过一个年，年后去上班。


## 初入职场，菜鸟快快长大
年后2月份来上班。最开始的工作只是很基础的用PHP索取数据渲染至HTML并输出的过程，还要经常使用Javascript完成前端动态交互。PHP和Javascript之前都没怎么接触过，所以需要一步步学习。

其实只要使用过函数式编程的语言（如C）便不难发现，PHP语言其实很好入门和进阶。但是Javascript则完全是不一样的逻辑，因为是EventDriven模式的，有着层层的回调和scope，而且代码执行流程也不是从上到下式的。所以在刚开始接触、使用JS的时候一定会遇到奇怪无比的坑。不过当你最终从坑里爬出来的时候，整个世界便又是另一番景象了。在这个过程中花时间学习和阅读Javascript相关的内容和资料的时间比正牌的后端PHP语言还要多，其实刚开始会有疑问，PHP毕竟是热门的后端语言，而JS只是很小众的前端交互助手而已，值得这样投入时间吗？在后来的工作中，以及nodejs火热起来之后我发现这些投入还是非常值得的，因为JS真的是我接触的第一个可以利用客户端的计算资源的语言，因为你用JS写好一个应用之后，剩余的运算执行是在客户端完成的，它使用的完全是客户端的计算资源！

现在回想一下，在编程工作起步的时候，最对的，也最受益无穷的选择便是坚持使用、查看英文资料。这真的是一个一本万利的事情。当时访问最多的编程问答网站就是Stackoverflow了。也许对一个有一定积累的人来说SO并没有多少更高的价值，但对于像我这样的菜鸟来说，SO简直就是高纯度的知识海洋。很多东西是对比着才能体会到其间的巨大差别的，当我总是查看英文的技术内容时，再回过头来偶尔看一下中文的问答或博客，简直如啖甘蔗：虽然看着内容很多，其实真正“香甜的甘蔗汁”只有那么一丁点，其余大部分全是废渣。

因为工作压力并不是很大，所以还利用晚上的时间参加了一门[Python公开课][python-on-coursera]的学习，一来可以练习英语（全英语视、听、作业），二来可以在学习编写小游戏的同时练习Python语言。就这样在持续学习了几个月后，自己在Python方面的实践经验也开始多起来。事实证明，很多看似无用、却由兴趣驱使着去做的事情，往往在后来可以大有所用，此事待后面详述。

日子就这样平淡着过去，转眼已是立夏。这时候公司开始想要做一款浏览器插件(Extension)，主要用来快速采集用户HTML中的产品信息，具体内容为跟用户所见的HTML交互并在用户人工识别页面信息后，友好地提取其中的产品信息，最后以表单的形式提交至我方服务器。这个[扩展][haitaobao-chrome-ext]在立项的时候是由另外一个同事负责开发的，但当时我意识到：1) 这是一个极好的了解浏览器API和练习用Javascript做App的机会；2) 那个同事的技术在我看来不足以做出我认为足够酷炫、友好的交互。我便先跟那位同事交流了我想独担这个任务的想法，取得他的同意后我向Boss主动请缨，Boss也欣然答应了。于是我接下了这个任务，开始了为期一个月的JSApp的编写工作。

JSApp的实现过程中最大的收获之一是接触到设计合理、博大精深的Chrome API。一直以来都以为Chrome只是快、好用而已，从未想到它竟然有如此多分层合理、设计合理的对外接口，简直就像是一个小型的操作系统。俗话说得好，经常使用设计良好的API，人生逼格也会静默升级呢(kidding)。也开始因为开发需要而接触到很多安全方面的issues，例如iFrame和它的父窗口之间只能发消息，但不能访问父窗口内的资源(window scope variables，like window.document)，例如[跨域访问][cors-upload-img]限制（虽然ChromeExtension运行空间直接允许跨域访问），例如[content security policy][chrome-app-csp]等。有时候会被这些看似太过苛刻的限制弄得f***这f***那的，这时候便会去试着了解一下他们这么做的原因或者好处。一来有助于缓解心中的"仇恨"(kidding)，二来也可以多学些东西。

JSApp实现过程中另一大收获便是Javascript的应用水平大大提高，开始逐渐喜欢上这门语言（有时候甚至会选择用它来练习算法）。在慢慢熟悉了JS的scope和回调之后，开始将一个个业务逻辑封装到独立的scope中，然后使用`Publish/Subscribe` Event&Msg-Driven的方式来连通各个scope，并驱动程序执行。这样做的好处有：1) 每个scope变得像是面向对象的类，封装了细节和实现; 2)因为使用消息驱动，所以代码解耦实现起来相对轻松。当然也是有坏处的，消息多了容易串，所以专门建立了一个manifest文件来快速查找、管理、记录这些消息/事件。

JSApp在完成发布后，Boss大加赞赏。对于一个刚开始只是做后端PHP开发的人员来说，出色地完成前端JSApp的实现也算是超出Boss的预期了。

Chrome扩展完成后，除了定期的维护升级外，基本就是日常的PHP端的HTML实现的工作分担一些，这样下来算是又闲下来了。我这人一旦闲下来就又开始不安分了。因为我们网站每天也有一万左右的访问量，所以有机会收集用户的请求信息。

网站后台当时有一个用户实时在线页面，可以查看哪些用户是最近登录的（实时在线只是个噱头，是根据session时间模拟的，毕竟HTTP无状态），也可以看到来自世界各地的用户请求。当时我突然想，要是能把我们客户在世界各地的分步情况实时地在一张平面的世界地图上展示出来，那该有多酷呀。说干就干，我利用业余时间开始做技术积累：用websocket实现服务器与客户端的实时通信，用python搭建websocket服务器，客户端直接使用JS(客户端需要支持websocket功能)；地图使用谷歌的，客户的访问IP转地理位置使用新浪的IPService接口，最后虽然做出来的样子不是很美观、也只能搭着梯子访问，但也算是达成目标了。

接下来就是公司技术博客的搭建了。因为公司运行的项目也有四年之久了，但却从没有一个比较系统的地方来介绍、讨论公司各个项目的架构、细节，以及服务器运维方面的东西，所以我就利用业余时间在我们公司的内网服务器上搭建了这个博客平台([截图][blog-10-jietu]，并邀请同事在上面分享日常的开发维护心得。


## 人事遽变，承担半边天

就在我过着按部就班毫无压力的日子的时候，公司却突然发生了三个月内的第二次人事遽变。第一次人事遽变，因为还有有过多年开发经验的同事扛着，也平稳度过了，然而第二次人事变动的主角就是这位同事。他要一走，说真的公司剩下的人里再没有一个能扛起这么多事情了：开发服务器的运维，生产环境十几台服务器的运维，网络爬虫系统的运维，网络搜索系统的运维，Python后端数据接口服务器的运维。当时一下子感觉天都要塌下来了！事实上后来和老板聊天时他说道，当时他已经开始考虑要使用外包了，他也根本没想到不起眼的我可以力顶压力，在把这些东西都一一搞定了。当然这是后话。

这位同事在半个月后就要离职，他当时也考虑过谁来接手项目的问题。当时公司其他搞开发的同事有用Python搞数据接口的，有用PHP搞前端页面实现和交互的。虽然他们有的有两年开发经验，但都是专注于自己的领域，在其他方向并没怎么拓展。而我恰恰在刚来的时候搞PHP页面实现和交互，在之后上Python公开课做过小项目(业余锻炼派上用场了)，再加上我为公司内网Linux服务器升级PHP及其扩展、搭建TODO App以及博客平台的经验，这位同事便推荐由我这个入职5个月的新人来接手这个项目，老板也是无可奈何地答应了（毕竟我们剩下的人中再无全能型技术人才了）。

而真正的压力、挑战和技术成长也是从这里开始。

首先是接手Python端数据接口项目。项目概述：组织内部自由业务，并为PHP提供Restful JSONFormat API。这个项目最大的挑战倒不是技术层面的，而是在内部业务方面既要知晓细节又要把控全局，那就没什么好说的，花大量时间阅读项目源码，理清脉络，在脑海中不断描述和完善整体结构。这个挑战持续了大概有两个月（和其他挑战并行），等到终于把大部分代码读过，又对项目使用的HTTP框架Flask有了进一步了解之后，就再无压力了。

接着是网络爬虫这块。项目概述：用Python脚本+Phantomjs解析HTML中的产品信息，用Redis做缓存，用HTTP在业务服务器和下载伺服器之间交换抓取任务和抓取到的产品信息。这里边最痛苦的莫过于一个个独立的任务队列了。队列任务流程：业务触发新的抓取请求，请求被投入抓取请求队列(业务服务器)；上步队列消费者索取下载请求，将其提交给下载伺服器接口；下载伺服器接收下载请求，将其放入待抓取队列；上步队列消费者索取请求，解析请求对应HTML中的产品信息，将解析所得产品信息投入信息回传队列；上步队列消费者索取产品信息，将其回传(提交)至业务服务器接口（流程完）。这里边的挑战有很多，1是队列很多，容易挂掉，所以用`supervisor`来保证队列生产消费者持续运行；2是由于队列过多带来的接口、流程复杂度的提高，因为项目一直也没有写测试代码，所以一个改动导致的bug很容易影响全盘的稳定运行。也是慢慢体会到没有测试代码所带来的代码改动时心里的没底以及随着改动带来的高额的人工debug的成本。不过好在各个环节都出几遍问题，自己再给调试好之后，整个项目的流程就彻底了然于胸了。在这个过程中稍微改动了一下用Redis做的队列的实现逻辑，为我们开发者提供了一些HTML内容索取接口和缓存服务，后期就只剩维护了。

## 力顶压力，独揽全局
接下来就是跟ElasticSearch（ES）相关的最后一个让人大头的项目，ES是一个可以高效率全文检索的搜索引擎。我在接手这个项目的时候使用的是v0.92版，对ES的接口用的PHP的第三方客户端。这个搜索引擎布置在生产环境，由两个物理节点组成了Cluster。但因为数据量大（800万+条信息）所以占用系统内存很多（为它分配了一半的物理内存）；而且每条信息排序的时候需要即时计算score，所以一有搜索请求过来，CPU的压力就会上来一点；再加上这个版本本身的bug导致它运行不够稳定，通常隔几天就会崩溃。当时我最怕的就是它崩溃了，本身手头上的开发任务就多，其他项目又都需要接手维护或管理，所以基本无心再顾忌这个搜索引擎了，只求使用上任负责人交给的死办法重启维护就行了。然而经过几个回合的崩溃、重启之后，我终于受不了了，开始慢慢安排，计划用当时最新的v1.4.1**无缝替换**掉线上运行的不稳定版本。接下来就是：用工作之余的时间不厌其烦地阅读有点不清晰的官方文档，在测试环境中下载、配置、安装和运行新版本，换用Python语言来使用ES更底层且灵活的API，为PHP写新的应用接口（Python提供），重建每条document在ES中的的检索/存储逻辑，把score的运算由即时分散到平时...所有这些准备工作花去了我一个月左右的业余时间，终于，最后成功地按照我的预期，在线上无缝地替换掉了旧版的ES。

从此时起，我便成为了公司里第一个也是唯一一个对当时所有项目都了然于胸的人！那种刚开始在大森林里只是负责种树苗，最后却得以爬上高山去俯瞰整个森林的感觉，真的太妙了，所谓“一览众山小”说的就是这种感觉。

也是从这时候起，Boss对我更加器重了（后来他还把他两室一厅的房子给我住，我住一间，另一件房间则允许我租出去，租金归我），自己做起其中的每个项目来，包括做出各种决策，也都更有方向感和把握了。

## 人事萧条，无心再战
然而从今年年初开始，随着公司开发人员的人才流失，到最后，基本就只剩我一个光杆司令了。Boss当时有些着急，直接允诺了不少的公司期权给我。然而我继续呆在这里，在可预见的未来，已经没有任何技术成长空间了。其实我最初来大城市的目的就是像牛人学习，结果到最后总是自己在推着自己向高处走，我觉得这样很不好，怕自己越来越狭隘。所以于最近（2015.06.28）向老板提交了辞职信，老板看我去意已决，再加上之前交流的时候有透露过这方面的想法，他也就没有再留我。当然公司这么多项目还是需要有人来接手的，所以我会再待一个月等人来接手。

## 心怀感恩，重新上路
其实如果没有这份工作提供给我一个进入编程领域的入口，我是不可能有今天的。我记得刚开始就是这种态度，老板在我刚开始进公司的时候就握着我的手说，谢谢你，加入我们！我当时的回答，也是现在的回答，是：也谢谢你，给我提供这样一个平台和机会。

[python-on-coursera]:http://blog.lyfing.co/2014/07/06/using-js-to-build-a-spaceship-floating-in-the-outer-space.html
[haitaobao-chrome-ext]:https://chrome.google.com/webstore/detail/%E6%B5%B7%E6%B7%98%E5%AE%9D-v2%E9%9D%9E%E7%9D%BF%E8%8E%AB%E6%80%9D/obnbgneldjmmpgkbnnbiiinijmiclpaa?hl=zh-CN
[cors-upload-img]:/2014/06/28/upload-browser-side-resources-by-using-xhr.html
[chrome-app-csp]:https://developer.chrome.com/apps/contentSecurityPolicy
[blog-10-jietu]:http://ww1.sinaimg.cn/large/6480dca9gw1etmhyd79i8j20k40l6jvv.jpg
