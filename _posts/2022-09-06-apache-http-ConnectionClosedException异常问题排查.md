---
layout: post
title: Apache Http ConnectionClosedException异常问题排查

tags: [Apache,httpasyncclient,ConnectionClosedException,tcpdump]
---

### 背景陈述
最近公司的一个应用，总是报`org.apache.http.ConnectionClosedException: Connection closed`Error日志。Error前后的逻辑是，本应用向远程调度系统发送HTTP请求，告知自己某项调度任务已经执行完成。但是偶尔在发送HTTP请求的时候会报这个Error.

本应用向远程系统发送HTTP请求，使用的是Apache的`httpasyncclient`包中的`HttpAsyncClient`完成的，简化后的Error日志如下
```
org.apache.http.ConnectionClosedException: Connection closed
    at org.apache.http.impl.nio.client.AbstractClientExchangeHandler.connectionAllocated(AbstractClientExchangeHandler.java:332)
    at org.apache.http.impl.nio.client.CloseableHttpAsyncClient.execute(CloseableHttpAsyncClient.java:92)
    
```

看现象是本应用的HTTP请求准备发出的过程中（可能在之前或者同时或者之后），底层复用的连接池中的TCP连接被关闭了，导致应用抛出了Connection closed的异常。


### 表面分析
应用的HTTP请求的header是这样设置的`Connection: Keep-Alive`，那就说明应用希望的是请求完成后不要关闭连接，但是啥原因导致连接被关闭了呢，又是TCP的哪一端发出的关闭连接的指令？

远程调度系统是一个老系统，维护的人也不太清楚发生了什么。目前来看仅通过上层的应用和功能分析，没办法快速深入分析和定位问题。

于是登录这个调度老系统，看了下他的TCP连接的状态汇总：
```
# cmd
netstat -ant | awk '{print $NF}' | grep -v '[a-z]'| sort | uniq -c
```

输出如下
```
1 CLOSE_WAIT
82 ESTABLISHED
6 LISTEN
104 TIME_WAIT
```

发现处于TIME_WAIT状态下的连接居多，然后又回顾了一下TCP关闭时4次挥手的流程图
![](/images/2022-09/tcp-connection-closed-four-way.png)

只有主动发起关闭的一方才会进入这个状态，而且处于这种状态的连接最多，初步判定就是服务端发起的关闭连接的请求。但这些分析仍然只是浮于表面，无法深入表面之下的核心原因。


正好因为之前开发PHP的DebugInWeb项目[`bughole`](https://github.com/HelloLyfing/bughole/wiki/How-to-debug-any-php-script-via-bughole-remotely) 时使用过`tcpdump`来抓取Debug通信协议的包，所以这个工具的使用经验可以派上用场了。

### Tcpdump抓包分析
Tcpdump是Linux系统下的一个抓包工具，可以将网络中传送的数据包完全截获下来提供分析。其抓包的原理是通过注册一种虚拟的底层网络协议来完成对网络报文(准确的说是网络设备)消息的处理权。当网卡接收到一个网络报文之后，它会遍历系统中所有已经注册的网络协议，例如以太网协议、x25协议处理模块来尝试进行报文的解析处理。系统在收到报文的时候就会给这个伪协议一次机会，让它来对网卡收到的报文进行处理，此时该模块就会趁机对报文进行窥探，也就是把这个报文完完整整的复制一份，并转发给抓包模块（感觉有点像系统设计里的Filter类）

由于两个系统的通信是基于7000端口，而且问题是偶现的，所以可以先通过监听并记录TCP协议的7000端口的TCP包信息，等到问题出现的时候再拿出这些包(packets)回溯现场。以下cmd是在本应用机器上执行的：
```
nohup tcpdump -nn -A -tttt tcp port 7000 > /tmp/tcpdump.txt &

# 参数解释
# nohup xx-cmd & 以后台程序运行
# -tttt  Print a timestamp, as hours, minutes, seconds, and fractions of a second since midnight, preceded by the date, on each dump line.
# -A  Print each packet (minus its link level header) in ASCII.  Handy for capturing web pages.
# -nn  Don't convert protocol and port numbers etc. to names either.
```

抓包几个小时后，打开抓包文件，发现了一个现象，TCP连接大多都是30s后被远程调度系统主动关闭的。下面贴出TCP连接建立、通信、关闭的相关日志。其中`192.168.103.136.7000`是远程调度系统的IP和端口，`10.32.142.236`则是本应用的IP。

TCP建立连接的日志如下：
```
# TCP的3次握手日志
2022-09-06 09:31:20.054553 IP 10.32.142.236.57838 > 192.168.103.136.7000: Flags [S], seq 914203194, win 29200, options [mss 1460,sackOK,TS val 2561453951 ecr 0,nop,wscale 9], length 0
E..<.E@.@.l:
....g....X6}.:......r..k.........
...........

2022-09-06 09:31:20.054746 IP 192.168.103.136.7000 > 10.32.142.236.57838: Flags [S.], seq 3996809024, ack 914203195, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 9], length 0
E..4..@.?.z...g.
...X...:w@6}.;..r..7.............

2022-09-06 09:31:20.054763 IP 10.32.142.236.57838 > 192.168.103.136.7000: Flags [.], ack 1, win 58, length 0
E..(.F@.@.lM
....g....X6}.;.:wAP..:.W..

```

TCP数据传输的日志如下（因为加了-A参数可以模糊看到明文传输的HTTP内容）：
```
2022-09-06 09:31:20.054943 IP 10.32.142.236.57838 > 192.168.103.136.7000: Flags [P.], seq 1:421, ack 1, win 58, length 420
E....G@.@.j.
....g....X6}.;.:wAP..:....POST / HTTP/1.1
Content-Length: 222
Content-Type: application/json; charset=UTF-8
Host: 192.168.103.136:7000
Connection: Keep-Alive
User-Agent: Apache-HttpAsyncClient/4.1.4 (Java/1.8.0_191)

{"methodName":"feedback","param":"{\"currentLogId\":1242911796,\"handleEndTime\":1662427880054,\"msg\":\"ok\",\"retry\":false,\"status\":\"SUCCESS\"}","parameterType":"com.ggj.platform.task.dc.core.job.model.JobRunResult"}

2022-09-06 09:31:20.055103 IP 192.168.103.136.7000 > 10.32.142.236.57838: Flags [.], ack 421, win 60, length 0
E..(..@.?.....g.
...X...:wA6}..P..<.;..

```

TCP关闭的日志如下：
```
2022-09-06 09:31:50.057120 IP 192.168.103.136.7000 > 10.32.142.236.57838: Flags [F.], seq 227, ack 421, win 60, length 0
E..(..@.?.....g.
...X...:x#6}..P..<.X..
2022-09-06 09:31:50.063784 IP 10.32.142.236.57838 > 192.168.103.136.7000: Flags [F.], seq 421, ack 228, win 60, length 0
E..(.I@.@.lJ
....g....X6}...:x$P..<.W..
2022-09-06 09:31:50.063953 IP 192.168.103.136.7000 > 10.32.142.236.57838: Flags [.], ack 422, win 60, length 0
E..(..@.?.....g.
...X...:x$6}..P..<.W..

```
这里其实有个疑问，TCP的挥手理应出现4条日志但只看到3条？[SO这里](https://stackoverflow.com/questions/51226711/tcp-4-way-close) 也有讨论，但不深入。如果有大神看到麻烦回复下发生了什么？（备注：已经知道`F`为Fin标识，`.`为ACK标识，附上前后半小时的抓包[完整日志](/static/other/2022-09/tcp-connection-packets-log.txt)）


从抓到的网络日志中可以看到几个现象：
1. 连接的建立由本应用发起；
2. 连接的关闭由远程调度系统发起；
3. 最后一次通信30s后，连接会被关闭，也就是idle 30s后会被关闭（可以从完整日志那里得到更多例证）；

### 基本定位问题
从上一步的抓包日志的现象中可以得出一些简单的结论：
1. HttpAsyncClient底层的连接池中的连接，己方不会主动关闭，而是由远程调度系统发起关闭；
2. 现象：当连接处于idle状态30s后会被远程关闭；猜想：远程调度系统一定设置了某种超时配置，这个配置的超时时间是30s；

有了猜想，就要进行验证。

我先clone了一下调度系统的代码，发现调度系统的server端使用的jetty。然后快速了解了下它使用的网络连接类`ServerConnector`，并在调度系统的server初始化配置中看到了这段代码：
```
Server server = new Server(new ExecutorThreadPool(threadPoolExecutor));

// connector
ServerConnector connector = new ServerConnector(server);
connector.setPort(port);
connector.setIdleTimeout(30000); // 这里的单位为ms，换算为秒正好就是30s！
server.setConnectors(new Connector[] {connector});
```

在顺藤摸瓜看下连接idle超时关闭的逻辑
```
protected void onIdleExpired(TimeoutException timeout) {
    Connection connection = _connection;
    if (connection != null && !connection.onIdleExpired())
        return;

    boolean output_shutdown=isOutputShutdown();
    boolean input_shutdown=isInputShutdown();
    boolean fillFailed = _fillInterest.onFail(timeout);
    boolean writeFailed = _writeFlusher.onFail(timeout);

    if (isOpen() && (output_shutdown || input_shutdown) && !(fillFailed || writeFailed))
        close(); // 关闭连接
    else
        LOG.debug("Ignored idle endpoint {}", this);
}

```

### 结论和总结

至此，Error日志的问题基本定位：本应用和调度系统通过HTTP1.1服务通信；本应用通过复用HTTP连接（连接池化）的方式跟对方通信；本应用期望通过“长连接”的方式保持跟对方的通信，但对方由于是HTTP SERVER必然考虑连接有效性，过段时间会主动关闭处于idle状态的连接。

经验总结：1）本应用或者类似应用，即期望通过复用HTTP连接而将HTTP连接池化的应用，应该做好对方主动关闭连接的异常情况，比如连接异常之后的重试要考虑下，继而也要考虑重试导致的Server端幂等问题；2）池化的HTTP连接，也可以考虑主动设置较短的idle回收时间，比如空闲5-10s后主动关闭连接，防止对服务端造成过多压力。


### 参考
 - TCP连接管理: https://www.cnblogs.com/idorax/p/6405172.html
 - TCP知识梳理: https://hit-alibaba.github.io/interview/basic/network/TCP.html
 - Tcpdump的用法: https://www.cnblogs.com/maifengqiang/p/3863168.html

