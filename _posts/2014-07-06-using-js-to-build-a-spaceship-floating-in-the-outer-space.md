---
layout: post
title: "MOOC课程小结：如何编写一个几乎无摩擦的太空飞船"
comments: true
tags: [Tech, Javascript, MOOC]
description: "MOOC课程小结：如何编写一个几乎无摩擦的太空飞船"
---

四五月份的时候曾抱着试一试的心态参加了一门MOOC课程[An Introduction to Interactive Programming in Python][mooc-python]，课程内容主要是通过编写小游戏来完成Python编程入门。

至于为什么要参加MOOC课程，又为什么选择了这一门呢？主要是MOOC这个话题太火了，弄得好像不参加就要当奥特曼了，所以希望通过参加一门简单的课程来了解MOOC；再者自己工作有闲暇时间，正好可以用业余时间学一学Python，也顺便学着编一些小游戏玩玩。

上完课拿到[证书][certification]后本想抽时间做个总结，却总是困于拖延症中无法自拔。自己这几天利用闲暇时间学了学HTML5的canvas，总算把当初在这门课程上学到的一丁点关于游戏开发的知识点 ——-穿行在摩擦系数极低的外太空中的太空飞船在飞行时的编程控制应该是怎样的—— 用javascript给实现了一下，就权当是总结吧。

想要玩一玩这个简易飞船的戳这里：[简易飞船demo][spaceship-link]

<!--more-->

###如何实现
关于这个飞船的飞行，当初最吸引我的一点是它在加速飞行时，加速方向和运动方向并不总是一致的，也就是说这个飞船在运行时是有"漂移"特效的，然后我就很好奇这种"漂移"特效是如何用编程来实现的。下面我就说一说这方面的事情。

####将飞船拟物化
也就是说从`面向对象`编程的角度来思考这个问题。

#####飞船的属性
拟物化的第一步是飞船在屏幕上得有一个起始点，在2d平面里可以直接用x（横坐标轴）和y（纵坐标轴）来标志起始位置即可。于是飞船有了第一个属性：当前坐标

    (x, y)

飞船从起始点(x1, y1)移动到(x2, y2)只能有一个原因，在某段时间内，它的速度不再为0(velocity)，然后用大家都熟悉的公式 `s2 = vel * time + s1` 便可以获得(x1,y1)到(x2, y2)的距离，所以我们需要赋予飞船第二个属性：速度

    vel速度(x、y代表速度分解到两坐标轴上的大小)

飞船具有速度从而开始运动，是因为它获得了某个方向的加速度(acceleration)。我们可以用公式 `vel_2 = acc * time + vel_1`  来获得飞船的速度，但前提是我们需要赋予飞船第三个属性：加速度

    acc加速度(x, y)

#####飞船的方法
想想看如果你坐在一架飘浮于外太空的飞船上时，你最需要飞船有什么功能？  

答案当然是可以挂档啦，要不然你就只能一辈子静止在那里看流星雨了。所以飞船应该要有一个挂档的方法(功能)

    startAcc() // 开始加速

Note: 正如科学家牛顿说过的，挂档的动作并不是改变飞船的速度(vel)，而是在改变它的加速度(acc).

如果你不想让飞船直接撞上附近某个星球，你就应该给它个挂空档的机会（即停止加速），要不然它真的会一直加速下去最终失控的（想想都可怕）

    stopAcc() // 停止加速

####万事具备，只欠挂档了 

 > 真的吗？

于是，当你按下加速键(↑)后，飞船开始加速，这个过程是发生在某个时间段内的，我们暂且就把这段时间定义为`1`吧（此处速度、时间等不再继承物理世界的单位，方便演示和计算）

在时间`1`内，君挂上档，飞船的加速度(`acc`)变化为：

    acc = (10, 10) // 任意数值，以飞船实际飞行效果调优为准

考虑到飞船初始时是静止的，所以飞船的速度变化为：

    newVel = acc * 1 + (0, 0) // newVel = (10, 10);

假设飞船初始位置为(0, 0)，那么飞船的位置变化为：

    newPos = newVel * 1 + (0, 0) // newPos = (10, 10);

此时如果君还没有挂空挡，那么在下一个时间`1`内，飞船的加速度(`acc`)、速度(`vel`)以及新位置(`pos`)将变为：

    acc = (10, 10) // 加速度此处设为常量（需要优化可以自己调优变动）
    newVel = acc * 1 + oldVel // newVel = (20, 20);
    newPos = newVel * 1 + oldPos // newPos = (20, 20);

此时需要设定一个定时器，每隔一小段时间(`interval`)把飞船的旧图像抹掉，再根据飞船的新位置将图像重绘一次，这样一来飞船看上去就好像动起来了。

可是，说好的漂移特效呢？

####飞船并不完整
要想有漂移，当然要有变向了。我们需要给飞船添加一个改变方向的方法，但前提是飞船要具有方向的概念（属性）。在飞船所处的2d世界，我们用**弧度**来表示它所朝的方向。
    
    // 飞船新属性
    ang = 0 // 0或者2π即表示水平向右的方向
    // 飞船新方法
    changeRotation() // 该方法的实质是改变飞船的 ang

添加新的属性和方法后，我们再来演示一遍在时间`1`内飞船各属性的变化（假设本时间段内飞船角度`ang`为 π/3，且处于加速状态，考虑到无摩擦时飞船在若干次加速后将因为速度过快而变得无法控制，我们在飞船更新速度时，为它添加一个速度磨损）
    
    // 角度
    ang = π / 3
    // 加速度
    acc = ( 10 * cos(ang), 10 * sin(ang) )
    // 速度
    vel = acc * 1 + oldVel
    // 摩擦导致的2%的速度磨损
    newVel = vel * 0.98 (小数大小以飞行体验调优为准)
    // 新位置
    newPos = newVel * 1 + oldPos

设置键盘
  
 - ←和→按下时调用`changeRotation()`方法来改变飞行角度
 - 按下↑时调用`startAcc()`方法启动加速
 - 释放↑时调用`stopAcc()`停止加速。

这次再试一试，飞船是不是开始漂移了？

Good luck :)

--------
**备注**：
 
 - MOOC公开课**Python交互式编程介绍**（飞船demo中所用图片素材也来自该课程）：https://class.coursera.org/interactivepython-004


[mooc-python]:https://class.coursera.org/interactivepython-004
[certification]:http://lyfing.qiniudn.com/docs/imgs/mooc/InteractivePython_Accomplishment_Statement.png
[spaceship-link]:http://lyfing.sinaapp.com/blog/demo/index/mooc-spaceship
