# 1 Memcached概述
## 1.1 Memcached是什么
Memcached是一个高性能、分布式的内存缓存系统，它通过在内存中缓存数据和对象来减少数据库的读取次数，从而提升动态数据库驱动型网站的访问体验。

Memcached本质上是一个基于String类型的键/值对的缓存系统。
![---1](/content/images/2018/06/---1.png)


## 1.2 Memcached的特性
***支持文本和二进制两种通信协议***
文本协议(`ASCII Protocol`)是Memcached一开始使用的协议，它简单可靠，我们可以通过telnet工具试用一下该协议：
```
telnet 127.0.0.1 11211
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.

# 客户端set命令
set testkey 0 60 5
hello

# 服务端响应内容
STORED

# 客户端get命令
get testkey

# 服务端响应内容
VALUE testkey 0 5
hello
END
```
为了提升客户端命令解析的速度，提高数据内容的可扩展性，Memcached在v1.3之后引入了Binary协议(`Binary Protocol`)。对这种协议感兴趣的可自行了解一下，此处不再细说。

***基于Libevent处理请求***
Libevent是一个提供异步事件回调的C程序。它支持`/dev/poll`, `kqueue(2)`, `POSIX select(2)`, `Windows IOCP`, `poll(2)`, `epoll(4)` 以及 `Solaris event ports`等各系统的调用，并对外提供了一套统一的API，使其在Server端跨平台的高并发应用场景中拥有独特优势。

***使用内存作为唯一存储介质***
是的，Memcached是一个只关注于缓存的系统，它不支持缓存的持久化，一切数据都只在RAM中存储。

***Key与Value长度有上限***
Key长度不能超过250个字节，Value长度则不能超过1MB.

***存储超限时使用LRU策略淘汰数据***
LRU(Least recently used)淘汰算法是实现简单、应用最为广泛的淘汰算法之一，后面我们会详细讨论它。

***实现逻辑：一半在服务端，一半在客户端***
比如Memcached分布式的实现，就是依靠客户端对Key的Hash散列实现的
![---1-1](/content/images/2018/06/---1-1.png)

# 2 Memcached的内存模型
下图是Memcached内存中的数据结构概览。
![---1-8](/content/images/2018/06/---1-8.png)
其中slab-class用来管理slab, 每个slab都是固定大小1MB的，但slab中真正用于存储数据(item)的chunk，它的尺寸在同一slab中是相等的。图中的item-hash-table用于存储key-item的映射关系。

那么slab到底是什么呢？Memcached在数据存储时为减少申请、释放内存产生的系统调用的次数（以提高效率）和释放内存产生的碎片， 每次向系统申请内存时会直接申请一个slab page size(1M)大小的内存空间，并将其分配给特定slab。
![---1-2](/content/images/2018/06/---1-2.png)
该slab则会将内存空间切分为size相同的块(chunk)
![---1-3](/content/images/2018/06/---1-3.png)
每个缓存的item，最终会落到和自己长度最接近的slab chunk中(item size <= trunk size)
![---1-7](/content/images/2018/06/---1-7.png)
每个slab大小都为1MB，为支持各种长度的item存储，多个slab在初始化其内部chunk时会按照增长因子(`factor`，可配置)依次递增，从而达到存储各种尺寸item的目的。
![---1-6](/content/images/2018/06/---1-6.png)

# 3 Memcached的数据结构
Memcached内部数据结构的核心是：LRU策略的缓存实现。这通常是由HashMap + LRU队列共同完成的。二者的关联关系如下图所示：
![---1-9](/content/images/2018/06/---1-9.png)

由于在执行每一次的set/get等命令时需要变更HashMap和LRU队列的顺序，在高并发场景下为保护数据的一致性，就必须对这两个数据结构进行加锁操作。当然了，加锁必然会带来并发性能的急剧下降。

我们来看一下一次客户端请求Memcached的完整工作流：
![---1-11](/content/images/2018/06/---1-11.png)
从上图可以看出对HashMap和LRU队列进行的操作加了全局锁，在高并发场景下，这种全局锁肯定是不可接受的，我们将试着提出一些优化的思路。不过在这之前，我们先简单了解一下这两种数据结构。

HashMap的结构图如下所示：
![---1-10](/content/images/2018/06/---1-10.png)
我们知道根据一致性哈希算法，给定一个key，它一定会落到HashMap(Table)中的某个Bucket中，多个落到同一Bucket中的key会被放入单向链表中去。

已知这个数据结构后我们会发现，给定单个key-item后，只需要完成对特定Bucket的锁定即可，无需锁定全局HashMap，这种锁便是分区锁。事实上Java语言中的ConcurrentHashMap就是通过这种锁实现的高并发操作。加锁的结构示意图如下：
![---1-12](/content/images/2018/06/---1-12.png)

接下来我们再了解一下LRU队列。LRU是一种淘汰机制，当给定空间不足时以最近最少使用为指标淘汰数据。其在操作系统内存管理、Innodb缓存管理中都有应用。
![---1-13](/content/images/2018/06/---1-13.png)
LRU的实现通常都是一个双向的链表，在不加锁的情况下对链表进行高并发操作容易造成各种问题（比如空指针），如下图所示。但加上全局锁则会对性能造成影响。![---1-14](/content/images/2018/06/---1-14.png)

对LRU的高并发操作的优化是一个持续的话题，本篇不再细讲，感兴趣的同学可以自行了解下。

# 4 Memcached的网络模型
为了支持高并发连接和跨平台部署，Memcached使用了Libevent来进行网络连接管理。

Libevent是一个提供异步事件回调的C程序。它支持 /dev/poll, kqueue(2), POSIX select(2), Windows IOCP, poll(2), epoll(4) 以及 Solaris event ports等各系统的调用，并对外提供了一套统一的API，使其在跨平台的高并发应用场景拥有独特优势。

说到select/poll，以及epoll/kqueue，就不得不说下他们的历史。从上世纪80年代开始，由于只有select和poll同步阻塞式的系统调用，Server端对客户端高并发网络连接的支持只能通过创建更多线程(池)或更强劲的机器性能实现。select和poll本质上相差不多，他们的时间复杂度都是O(n).
![---1-15](/content/images/2018/06/---1-15.png)

不过当Unix系统引入了epoll和kqueue等系统调用后，Server端对高并发网络连接的支持困境得到了极大优化。二者的时间复杂度都是O(1)!
![---1-16](/content/images/2018/06/---1-16.png)

Libevent正是上面所述的系统调用的集大成者，它封装个底层操作系统的系统调用函数并对开发者提供了统一的API，也难怪它被广泛地应用在跨平台的高并发场景。
![---1-17](/content/images/2018/06/---1-17.png)
# 5 Memcached vs Redis
谈到Memcached的对比，就不得不拿老对手Redis来进行一番比较。以下是二者的对比
![---1-18](/content/images/2018/06/---1-18.png)
从对比中可以看出，Redis在不断迭代中已经成长为一个更全能的存储DB，缓存服务只是它功能的一部分(虽然仍是主要部分)；Memcached则始终关注String缓存这一功能，并且借由开源工具弥补了自己在集群、备份、主从等方面的短板。

二者并没有完全的孰优孰劣的情况，在实战中应根据实际场景具体问题具体分析，毕竟最合适的才是最好的。
