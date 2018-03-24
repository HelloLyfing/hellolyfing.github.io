---
layout: post
title: Python爬虫 实现从糗百上多线程抓取内容
description: "用Python写的爬虫，可以从糗百上抓取所有文章，并下载相关附图（如果附图的话）"
tags: [Tech, Python, 多线程, 爬虫, 糗事百科, BeautifulSoup]
comments: true
share: true
---

最近参加一家公司的远程笔试，其中的一道题目是：

 > 写一个简单的爬虫，把糗事百科今天被顶超过5000的帖子爬出来，注意考虑性能和图片显示。

当时一看很感兴趣，因为看到这道题目后思路很清晰，而且我大学时周围好友都爱看糗百，所以做点有关他们喜欢的产品的信息抓取还是挺有趣的。

好的，闲话就到这里，下面进入正题。

<!--more-->

## 1. 思路

### 1.1 单线程 or 多线程

**单线程**按序逐一抓取。这种思路下的实现方式是：
 
 1. 获取糗百所有可供抓取的页面URL，然后把他们放到一个列表里
 2. 从列表中取走一条页面URL，将该URL指向页面中的所有糗百文章解析出来
 3. 如果文章有附图，则下载至指定目录
 4. 将第2步获得的若干糗百文章追加至一个xml文件中
 
点评：单线程无法充分利用机器的CPU资源和带宽，性能低下，不予考虑

**多线程**乱序抓取。这种思路下的实现方式是：

 1. 创建两个[同步队列][tong_bu_queue]`page_q`和`pic_q`，`page_q`存放页面URL，`pic_q`存放图片URL
 2. 获取糗百所有可供抓取页面的URL，将这些URL添加到`page_q`队列
 3. 开辟多条**抓取解析页面文章的线程**，每条线程的具体工作是：

     - 从`page_q`队列取走一条URL，解析其指向的页面中的糗百文章
     - 将这些文章内容追加到xml文件中(同步访问)
     - 如果文章有附图，则将该附图的链接URL放入`pic_q`
 
 4. 开辟多条**下载图片的线程**，每条线程的具体工作是：

     - 从`pic_q`队列取走一条图片URL，将其命名为`idxxxxx.jpg`并下载到指定目录 

点评：可以充分利用CPU资源及带宽，选择该条思路进行

### 1.2 如何提取页面内容

 - 思路1：通过正则表达式匹配，然后提取有用信息
 - 思路2：通过第三方HTML内容提取工具提取有用信息

点评：思路2具有更高的扩展性、容错性，选择思路2

## 2. 实现

### 2.1 多线程实体

本文设计了两个多线程类，他们都继承自`threading.Thread`，这两个类是：  
 
 - `class QiubaiReader(threading.Thread)`
 - `class PicDownloader(threading.Thread)`

类`QiubaiReader`要做的工作和本文第1部分 **1.1** >> **多线程** >> **抓取解析页面文章的线程**内容一致，以下是它的执行逻辑：

    """ 类 QiubaiReader 说明
    糗事百科内容的消费者，也是糗事百科文章图片的生产者
    消费者：从 pageQueue 里读取一个页面URL，解析该页面所有糗百文章，并将这些文章存储到xml文件中；
    生产者：在解析页面时，如果某篇文章附带图片，则把该图片的URL放入 picQueue ,等待图片类的消费者来处理.
    """
    runFlag = 1                                                    #停止线程的开关

    def __init__(self, pageQueue, picQueue, pathDict):
        ...

    def fetchContent(self, pageUrl):
        """
        每一条糗百，我们需要取出其中的三条信息：
        1. 该条糗百的ID
        2. 该条糗百的正文
        3. 该条糗百的图片链接(可能为空，非空则等待下载)
        爬取糗百的步骤是：
        1. 获得该条糗百的整个大<div />块，我们声明 div_dad 变量代表这个大<div />块
        2. 通过查找当前投票数的<div />相关值来判断是否继续，如果投票数大于5000，则继续
        3. 在大<div />块的首行，截取该条糗百的文章ID号（每条糗百都是一篇文章，通过文章ID可以获取文章和图片的链接）
        4. 在大div块中，找出带有糗百正文的<div />块
        5. 将上面提到的三条信息写入xml文件
        """
        ...

    def writeContent(self, list):
        """
        将list中包含的糗百文章格式化并一次性插入到xml文件中
        list中包含有某个页面的所有糗百文章（一般是20条）
        list结构为：
        [
            {   'id':       qiuID1,
                'content':  qiuBaiText,
                'picURL':   picURL },
            ...
        ]
        """
        ...

    def run(self):
        while not self.pageQueue.empty() and self.__class__.runFlag > 0:
            #糗百页面消费者
            pageUrl = self.pageQueue.get()
            qiuBaiList, picDictList = self.fetchContent(pageUrl)
            if len(qiuBaiList) >= 1:
                self.writeContent(qiuBaiList)
            #糗百图片生产者
            if len(picDictList) >= 1:
                for item in picDictList: self.picQueue.put(item)

        #如果两个队列都为空，则线程退出，并通知图片下载线程也退出
        if self.pageQueue.empty() and self.picQueue.empty():
            ...

类`PicDownloader`要做的工作和本文第1部分 **1.1** >> **多线程** >> **下载图片的线程**内容一致，以下是它的执行逻辑：
    
    # 类 PicDownloader 说明
    runFlag = 1                                                            #线程停止开关

    def __init__(self, queue, pathDict):
        ...
    def downloadPic(self, picDict):
        ...
    def run(self):
        while self.__class__.runFlag > 0:
            while not self.queue.empty():
                picDict = self.queue.get()
                self.downloadPic(picDict)
            time.sleep(1)                                                  #如果图片URL队列为空，则等待一秒


### 2.2 线程安全队列

本文涉及到的两个队列`page_q`和`pic_q`，一个用来存取页面URL，另一个用来存取图片URL，两队列都面临着多线程同步存取的问题，而这则是所有的"生产者-消费者问题"必须解决的问题。

幸运的是，我们用的是Python！  

Python已经为我们提供了一个线程安全队列：[Queue][tong_bu_queue]，它为多个"生产者-消费者"提供了安全同步队列。引用官方的一句话便是：

 > The `Queue` module implements multi-producer, multi-consumer queues. It is especially useful in threaded programming when information must be exchanged safely between multiple threads.

而且`Queue`的创建、使用也极为轻便    
创建
    
    import Queue
    queue = Queue.Queue()

使用

    item_1 = queue.get()     # queue.get() => 从队列中移除一个item并返回该item
    queue.put(item_2)        # queue.put() => 往队列中添加一个item

对`Queue`更高要求的操作与使用，请查看[官方文档][tong_bu_queue]。

### 2.3 HTML内容提取

该部分内容，其实是对类`QiubaiReader`中的`fetchContent(self, pageUrl)`方法的解读。从HTML中获取内容时，我们需要借助第三方开源工具[BeautifulSoup][bs4_page](看最下方应用程序信息)

为了便于升级改动，我们为类`QiubaiReader`声明一个类成员变量`argsDict`，用来统一糗事百科HTML页面源码中的一些关键性的标记及属性

    argsDict = {
            'pageEncoding'  : 'utf-8',                                 #糗百html的编码格式
            'dadClassAttr'  : 'block untagged mb15 bs2',               #某条糗百整个大<div />块的class属性
            'contClassAttr' : 'content',                               #某条糗百的正文所在<div />块的class属性
            'picClassAttr'  : 'thumb',                                 #包含图片的<div />块的class属性
            'voteClassAttr' : 'bar',                                   #包含投票数的<div />块的class属性
            #包含糗百ID的那一行的id号前的前缀，例如：'qiushi_tag_55611097'
            'idLinePreStr'  : 'qiushi_tag_',                           
            #某条糗百只有点赞数超过该值，才进行收录。题目要求该值为5000，本人感觉偏高，故将其改成了2000
            'validCountNum' : 2000,                                    
    }

糗百每一个页面会包含20条糗百文章，读者可以[点此查看][qiubai_part1]其中某条糗百文章的HTML源码及其标记结构。

我们会发现糗百文章的HTML标记结构如下（我们姑且把如下`<div />`块称作`文章的<div />`块吧）：

    <div class="block untagged mb15 bs2" id='qiushi_tag_idxxxxxx'>
        <div class="content" title="2014-01-14 16:33:29">
            糗百正文
        </div>
        <!--除非文章配有图片，否则下面这个div不会出现-->
        <div class="thumb">
            <a href="/article/url..." target="_blank" onclick="some js">
                <img src="http://the/pic/URL" alt="图片描述" />
            </a>
        </div>
    </div>
    ...
    一个页面中会有20个上述结构出现，也就是20条糗百文章
    ...

以下的任务是，解析给定页面中的所有糗百文章，获取它们的点赞数、ID、正文以及图片链接（如果有的话）。这些解析工作需要借助`BeautifulSoup`工具来完成。
 
**step 1:**我们首先引入`BeautifulSoup`，并实例化一个可操作的HTML结构体：
    
    import urllib2
    from bs4 import BeautifulSoup
    #获取给定页面pageURL的HTML源码
    pageCont = urllib2.urlopen(pageURL).read().decode(self.argsDict['pageEncoding'])
    #将HTML源码传递给BeautifulSoup，实例化一个它的对象
    soup = BeautifulSoup(pageCont)

**step 2:** 获得给定页面的所有20个上述的`文章<div />`块

HTML是标记性语言，即使其中的`<div>`标记(tag)出现了很多次，而且分布杂乱，但我们可以根据一个`<div>`标记的多个属性来唯一确定某类/某个标记。
例如`文章<div />`块的属性是：

    <div class="block untagged mb15 bs2" id='qiushi_tag_idxxxxxx' ></div>
    即：
    class = "block untagged mb15 bs2"
    id = "qiushi_tag_idxxxxxx"

`文章<div />`块的`id`属性不确定，但它的`class`属性是确定且唯一的，我们就使用它的`class`属性来找到这20个`文章<div />`块，并把它们保存到一个`list`中
    
    articles_div_list = soup.find_all('div', attrs={'class': 'block untagged mb15 bs2'})

**step 3:**接下来，我们遍历`articles_div_list`，并从中解析出我们需要的糗百信息
    
    for div_article in articles_div_list:
        ...
           
step 3.1 获得点赞数（[点此查看][div_mark_vote]点赞内容所在`<div>`标记）

    div_vote = div_article.find('div', attrs={'class': 'bar'})  #用给定的属性键值对（class='bar')查找某个标记（tag）
    upCount = div_vote.a.get_text()                      #通过 标记.字标记.get_text() 方法获得字标记的text

step 3.2 获得ID（[点此查看][div_mark_id]ID内容所在`<div>`标记）
    
    idLine = div_article.attrs['id']  #想要获得某个标记（tag）的属性，可以直接查字典一样，此处key为某个属性的name

step 3.3 获得正文（[点此查看][div_mark_cont]正文内容所在`<div>`标记）

    div_cont = div_article.find('div', attrs={'class': 'content'})
    qiubai_cont = div_cont.get_text()                #通过标记的 get_text() 方法获得该标记的text

step 3.4 获得配图的URL（如果有的话。[点此查看][div_mark_pic_url]配图URL内容所在`<div>`标记）

    div_pic = div_article.find('div', attrs={'class': 'thumb'})
    if div_pic:
        #想要获得某个标记（tag）的子标记的子标记...的属性，可以直接通过 .(英文句点) 索引至该标记，然后像查字典一样查找即可
        picURL = div_pic.a.img['src']

### 2.4 存储内容

因为糗百内容要存储到xml文档中，我们在这里还要使用Python自带的操作XML的包：`xml.etree.ElementTree`

**step 1：**我们首先创建一个用于存储糗百的xml文档：
    
    fo = open('qiubai.xml', 'w')
    fo.write('<?xml version="1.0" encoding="utf-8"?>\n<ROOT></ROOT>')
    fo.close()

此时的xml文件看起来应该是这个样子：
    
    <?xml version="1.0" encoding="utf-8"?>
    <ROOT>
    </ROOT>

**step 2：**给`qiubai.xml`添加一条糗百内容

step 2.1：获得`qiubai.xml`文档的根节点
    
    import xml.etree.ElementTree as ET
    tree = ET.parse('qiubai.xml')       #也可以给 parse() 传递文档路径
    root = tree.getroot()

step 2.2：为根节点`root`添加一个子节点`QiuBai`，并设置该子节点的各个属性
    
    qiubai = ET.SubElement(root, 'QiuBai')
    qiubai.set('id', 'idxxxxxx')
    qiubai.set('picURL', 'http://here/is/pic/url.jpg')
    qiubai.text = '此处为糗百正文...'

step 2.3：将添加了新内容的`root`保存到文档
    
    tree = ET.ElementTree(root)
    tree.write('qiubai.xml', encoding='utf-8', xml_declaration=True)

此时的xml文档看起来应该是这样子：
    
    <?xml version="1.0" encoding="utf-8"?>
    <ROOT>
        <QiuBai id='idxxxxxx' picURL='http://here/is/pic/url.jpg'>
            此处为糗百正文...
        </QiuBai>
    </ROOT>

注意事项：如果你使用的是UTF-8格式保存xml文档，那你需要注意：xml文档规范并不支持所有的UTF-8支持的字符，也就是说有些UTF-8支持的字符在xml文档中是不受支持的，如果你坚持写入，则在再次读取xml文档是会出错。

关于过滤xml不支持字符的内容，请参看源码 `QiubaiReader.py` >> `def replaceHellWord(text)`方法

## 3. 源代码

到[这里][github_src]查看源码，或者直接[点我][downl_src]下载源码。

应用程序信息：

 - 本程序在Window7平台下开发完成并测试通过；在CentOS 6.3下测试通过
 - Python `2.7.5 [MSC v.1500 64 bit (AMD64)]`
 - xml.etree.ElementTree `1.3.0` 

    - 键入 `import xml.etree.ElementTree`； `ElementTree.VERSION` 查看
 
 - Beautiful Soup `4.3.2`                           

    - 键入 `import bs4`；`bs4.__version__` 
    - [bs4下载链接][bs4_downl]

[github_src]:https://github.com/HelloLyfing/Tiny_Projects/tree/master/WebSpider
[downl_src]:https://github.com/HelloLyfing/Tiny_Projects/raw/master/WebSpider/web_spider.tar
[bs4_downl]:http://www.crummy.com/software/BeautifulSoup/bs4/download/4.3/
[bs4_page]:http://www.crummy.com/software/BeautifulSoup/
[tong_bu_queue]:http://docs.python.org/2/library/queue.html

[qiubai_part1]:https://code.csdn.net/snippets/156535/master/qiubai_content/raw
[div_mark_vote]:https://code.csdn.net/snippets/156687/master/div_mark_vote/raw
[div_mark_pic_url]:https://code.csdn.net/snippets/156693/master/div_mark_pic_url/raw
[div_mark_id]:https://code.csdn.net/snippets/156695/master/div_mark_id/raw
[div_mark_cont]:https://code.csdn.net/snippets/156709/master/div_mark_cont/raw
