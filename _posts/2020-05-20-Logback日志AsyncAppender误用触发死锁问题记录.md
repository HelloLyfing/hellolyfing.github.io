---
layout: post
title: Logback日志AsyncAppender误用触发死锁问题记录
---

* unOrder TOC
{:toc}

# 一、背景描述
我司的某个应用使用的Logback输出日志，之前应用的info日志是通过`RollingFileAppender`输出的。
其中RollingFileAppender的工作流程图示如下：
![](/images/2020-05/AsyncAppender死锁问题记录-RollingFileAppender工作原理图示.png)

后来我增加了一些业务日志，并随手把Appender改成了`ch.qos.logback.classic.AsyncAppender`，出于两方面的考虑：
 - 1）新增了一些info日志后，希望日志输出这块不会受影响；
 - 2）异步输出info日志对应用的性能影响更小，所以就随手改了下。

改动后的配置如下：
```
<!--异步输出-->
<appender name="ASYNC_INFO" class="ch.qos.logback.classic.AsyncAppender">
    <!-- 即便异步队列满了也不要丢弃info及以下的日志 -->
    <discardingThreshold>0</discardingThreshold>
    
    <queueSize>512</queueSize>
    <!-- 在本机和测试环境，覆盖默认队列长度为10 -->
    <springProfile name="local,test">
        <queueSize>10</queueSize>
    </springProfile>

    <appender-ref ref="INFO"/>
</appender>
```

顺便简单了解了下AsyncAppender的工作原理，其实就是把日志事件(LogEvent)暂存到`ArrayBlockingQueue`中，再由单一的Worker线程从队列中取出元素并执行真正的输出工作。

AsyncAppender工作流程图示如下，图中的AsyncThread2线程，即为上面提到的Worker线程
![](/images/2020-05/AsyncAppender死锁问题记录-AsyncAppender工作原理图示.png)

然而这段代码发到测试环境后，却发现应用隔一段时间之后就开始出现无响应的情况。本应用是Dubbo RPC服务，无响应的具体表现为：消费者调用本服务时会出现100%超时并100%失败，100%超时说明服务仍可以建立连接，只是迟迟不响应。

# 二、寻找原因
第一反应是查看下线程的堆栈信息。

步骤1：登录到该应用的应用服务器上，先找到对应的Java进程id
```
# 参数解释
# l：打印执行的Jar文件名
# v：打印运行时的执行参数
jps -lv
```

步骤2：通过jstack命令Dump应用的线程运行情况
```
jstack $pid
```

查看线程Dump日志后发现，200多个Dubbo服务线程在打印日志时（`调用logger.info()`），因为执行了AsyncAppender.ArrayBlockingQueue的put操作被阻塞住了（其实是Dubbo服务线程在`notFull`的Condition上被阻塞住了），总之就是这个日志事件队列已经满了。
```
"DubboServerHandler-172.20.32.138:20880-thread-300" #662 daemon prio=5 os_prio=0 tid=0x00007f717426c800 nid=0x2bd waiting on condition [0x00007f70e603a000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00000000e21d3328> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
	at java.util.concurrent.ArrayBlockingQueue.put(ArrayBlockingQueue.java:353)
	at ch.qos.logback.core.AsyncAppenderBase.put(AsyncAppenderBase.java:160)
	at ch.qos.logback.core.AsyncAppenderBase.append(AsyncAppenderBase.java:148)
	at ch.qos.logback.core.UnsynchronizedAppenderBase.doAppend(UnsynchronizedAppenderBase.java:84)
	at ch.qos.logback.core.spi.AppenderAttachableImpl.appendLoopOnAppenders(AppenderAttachableImpl.java:48)
	at ch.qos.logback.classic.Logger.appendLoopOnAppenders(Logger.java:270)
	at ch.qos.logback.classic.Logger.callAppenders(Logger.java:257)
	at ch.qos.logback.classic.Logger.buildLoggingEventAndAppend(Logger.java:421)
	at ch.qos.logback.classic.Logger.filterAndLog_1(Logger.java:398)
	at ch.qos.logback.classic.Logger.info(Logger.java:583)
```

可是为什么队列会满呢？不是有一个AsyncAppender.Worker线程在执行从队列取元素的操作吗(`queue.take()`)？即便Worker处理的较慢导致队列偶尔满了，但也不应该导致put线程一直阻塞，继而导致Dubbo服务线程 100%无响应吧？于是我好奇的看了下这个Worker线程的状态
```
"AsyncAppender-Worker-ASYNC_INFO" #51 daemon prio=5 os_prio=0 tid=0x00007f71b44df000 nid=0x62 waiting on condition [0x00007f712428d000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00000000e21d3328> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
	at java.util.concurrent.ArrayBlockingQueue.put(ArrayBlockingQueue.java:353)
	at ch.qos.logback.core.AsyncAppenderBase.put(AsyncAppenderBase.java:160)
	at ch.qos.logback.core.AsyncAppenderBase.append(AsyncAppenderBase.java:148)
	at ch.qos.logback.core.UnsynchronizedAppenderBase.doAppend(UnsynchronizedAppenderBase.java:84)
	at ch.qos.logback.core.spi.AppenderAttachableImpl.appendLoopOnAppenders(AppenderAttachableImpl.java:48)
	at ch.qos.logback.classic.Logger.appendLoopOnAppenders(Logger.java:270)
	at ch.qos.logback.classic.Logger.callAppenders(Logger.java:257)
	at ch.qos.logback.classic.Logger.buildLoggingEventAndAppend(Logger.java:421)
	at ch.qos.logback.classic.Logger.filterAndLog_2(Logger.java:414)
	at ch.qos.logback.classic.Logger.info(Logger.java:587)
	at com.ggj.user.commons.utils.XconfReaderUtils.safeGetConf(XconfReaderUtils.java:31)
	at com.ggj.user.commons.filter.log.LogAlarmFilter.needIgnoreMessage(LogAlarmFilter.java:188)
	at com.ggj.user.commons.filter.log.LogAlarmFilter.decide(LogAlarmFilter.java:80)
	at com.ggj.user.commons.filter.log.LogAlarmFilter.decide(LogAlarmFilter.java:26)
```
结果发现，这个Worker线程，竟然也因为需要打日志，而往队列执行put操作时被阻塞了（可以看到Dubbo服务线程和Worker线程都在等待`0x00000000e21d3328`这个锁）。也就是作为日志事件的队列消费者的Worker竟然也变成了生产者，而且意图往已经满了的队列put元素并被阻塞了，最终这个队列唯一的一个消费者线程阻塞住了。没有了消费者，队列永远是满的状态，出现了死锁，所以整个Dubbo服务一直被阻塞了。

# 三、死锁原因
仔细看上面的Worker线程堆栈，Worker线程竟然也在调用`logger.info`方法从而使自己变成了日志事件的生产者，继而引发了队列的死锁问题。

Worker线程中`logger.info()`函数的具体调用方为：LogAlarmFilter。该Filter是我司的一个业务监控Filter，其目的为：将收集到的应用的Error日志异步发送到钉钉群。
它的实现原理为：伪装自己为一个`AbstractMatcherFilter`（我们熟知的LevelFilter就是该类的子类），并且在应用启动的时候，注册到所有logger(包含root)上去，以实现Error日志的收集和报警。

问题是：LogAlarmFilter作为`logger.info()`日志事件的处理者，却在自己代码内再次调用了该方法生产新的日志事件，这么做其实很不合理，一来容易导致隐式的递归调用；二来也造成了上述死锁的问题发生。

# 四、优化解决
通过前面的讨论可以发现，Worker线程的执行链路，最好不要或者干脆禁止通过`logger.xxx()`方法打印日志，以避免造成该线程变为日志事件的生产者，而去竞争队列锁并出现可能的递归调用。

**进一步延伸下，业务方自己实现的Logback的组件（包括Logback自带的组件），通常都是`logger.xxx()`函数的处理链路中的一环，其处理逻辑中应该禁止调用`logger.xxx()`函数输出日志信息。**
但这些组件确实也有需要向外界（比如向开发者）传递程序运行时状态或数据的需求（比如请求走到了哪里、参数为何、输出组件的Error异常等）。

针对这种需求，其实Logback早有解决方案：通过`ContextAwareBase.addStatus()`系列方法，向外界输出程序运行时状态或数据，业务方再通过`StatusListener`即可监听Logback内部组件发出的message。Logback的大部分组件都继承了ContextAwareBase，所以可以直接使用该类提供的方法。具体使用方法和原理请自行搜索查找，此处不再细讲。

话说回来，上面的LogAlarmFilter有两个优化点需要做：
 - 1）由于自己属于`logger.xxx()`函数的处理链路中的一环，需要打印日志的地方，改用：`ContextAwareBase`相关方法即可，例如：`addInfo()`；
 - 2）为了实现：将收集到的应用的Error日志进行异步钉钉群通知的需求，其实把LogAlarmFilter设计为Appender类更好。因为Appender就是作为一个输出终端来设计的，而钉钉群通知，无疑就是一个类似文件和控制台一样的"终端"。

文章最后，解释下`discardingThreshold`参数的含义，以及如何合理配置。
以下是我的生产环境配置，其中队列queueSize设置为512，丢弃阈值设置为30
```
<!--异步输出-->
<appender name="ASYNC_INFO" class="ch.qos.logback.classic.AsyncAppender">
    <queueSize>512</queueSize>
    <discardingThreshold>30</discardingThreshold>
    <appender-ref ref="INFO"/>
</appender>
```

感觉直接看下源代码（为便于阅读适当调整了源代码）更有意义
```
// discardingThreshold设置默认值的逻辑。如果未指定该值，则默认值为队列size的1/5
if (discardingThreshold == UNDEFINED)
    discardingThreshold = queueSize / 5;

// 日志事件输出逻辑
@Override
protected void append(E eventObject) {
    // 如果日志级别在Info及以下为true
    boolean isDiscardable = event.getLevel().toInt() <= Level.INFO_INT;
    
    // 如果队列可用size < discardingThreshold为true
    boolean isQueueBelowThreshold = blockingQueue.remainingCapacity() < discardingThreshold;
    
    if (isQueueBelowThreshold && isDiscardable) {
        // 如果上述两个条件都满足, 直接丢弃日志事件，不处理
        return;
    }
    
    preprocess(eventObject);
    putIntoQueue(eventObject);
}
```

# 参考资料
 - 图中部分图片来源: http://codetd.com/article/9383030

