---
layout: post
title: "关于Python的编码、乱码以及Unicode的一些研究"
comments: true
tags: [Tech, Python, Unicode, Language Study]
comments: true 
description: "关于Python编码、解码、乱码以及Unicode的一些研究"
---

最近接触Python比较多，尤其是在命令行(Terminal)下进行的局部代码测试有很多。而个人编写的代码通常是以`UTF-8`格式存储的，这在Linux下的Terminal上还好一些（它的编码默认的就是中文`UTF-8`），要想打印包含中文字符的变量值，基本不会出现乱码情况。但如果是在Windowss下的cmd上进行测试，则相对就要痛苦一些，因为Windows中文环境下cmd的默认编码是`GBK`。

所以本人为了在测试时能正常打印出中文字符（经常Linux、Windows两边跑），对Python字符编码的情况还是颇下了些功夫的。下面就是我做的一些小研究，希望能对和我当初一样迷茫的读者起到些帮助作用。

<!--more-->

### 1. ASCII, Python默认的编码

在Python中，当源代码被读取进行语法校验时，会将源代码中的字符从声明的编码转换成`Unicode`类型，等到语法校验通过后，再将这些字符转换回初始的编码。

在Python环境下，源文件中如果没有声明编码，则将其编码设置为默认的`ASCII`，这可以通过下面这段代码检验：

    >>> import sys
    >>> sys.getdefaultencoding()
    'ascii'

所以在编写源代码文件`xxx.py`时需要注意：

 - *如果源代码没有声明编码格式，则Python在语法校验期间(compilation)使用默认的ASCII编码*
 - *如果源代码没有声明编码格式，但却在源码中使用了非ASCII字符(Non-ASCII character)，则程序在编译期间抛出`SyntaxError`异常，编译不被通过*
 - *如果源代码声明了Python不支持的编码格式，则程序将在编译期间抛出异常
 - *源代码文件编码的声明*

     - *查看如何在源代码中声明编码格式[PEP-0263][pep-0263]*
     - *查看Python支持的[编码格式][supported-encoding]*


### 2. 命令行与Python——乱码篇
在命令行下的Python Shell中进行小范围测试，或者在命令行中运行Python代码时，总会遇到各种各样的乱码问题。为了搞清楚乱码的原因，我们首先来看这样一个例子。

例子：文件到底长什么样？

首先, 在某目录(假设D:\Tem\)下创建两个文本文件`1.ini`、`2.ini`，两个文件中都只写入下面这行内容。不同的是，`1.ini`以UTF-8格式保存，`2.ini`以GBK格式保存
    
    我是Lyfing.Loo，来自中国 

接着, 在命令行下，`cd`至该目录，键入`Python` 进入Python Shell, 读取两个文本文件中的内容，并打印出来

    >>> str1 = open('1.ini','r').read()     
    >>> str2 = open('2.ini','r').read()    
    >>> str1    
    '\xef\xbb\xbf\xe6\x88\x91\xe6\x98\xafLyfing.Loo\n'    
    #以16进制形式查看1.ini(UTF-8)    
    efbb bfe6 8891 e698 af4c 7966 696e 672e 4c6f 6f0d 0a    
    >>> str2   
    '\xce\xd2\xca\xc7Lyfing.Loo\n'     
    #以16进制形式查看2.ini(GBK)    
    ced2 cac7 4c79 6669 6e67 2e4c 6f6f 0d0a    

*其中`0d0a`是换行符，0d——回车符号——"\r"，0a——换行符号——"\n"*

对比一下`str`内容和16进制下查看到的相应文本中的内容，我们不难发现，Python在读取文件时，除了辨认出ASCII编码范围内的字符之外，例如L(4c)、g(67)和o(6f)等，其他非ASCII的字符一律仍按16进制(由`0` `1`二进制位组成)的原始编码储存，例如 我(efbbbf, utf-8)和我(ced2, gbk)。等到需要将变量值（字符串）打印到控制台，或者写入文件时，这一串原始编码便会原封不动地被提交给相应程序。

仍是上面的例子，当我们想要打印`str1`和`str2`的值时，我们看看会发生什么：

    >>> print str1
    锘挎垜鏄疞yfing.Loo
    >>> print str2
    我是Lyfing.Loo

我们看到，`str1`出现了乱码，而`str2`则正常打印出了中文字符。这是为什么呢？还是上面那句话：

 > 等到需要将变量值（字符串）打印到控制台，或者写入文件时，这一串原始编码便会原封不动地被提交给相应程序

当需要打印`str1`和`str2`的值时，Python并不做任何编码转换的处理，它只是原封不动地将原始编码提交给了控制台，由相应的控制台（Windows下的cmd/Linux下的Terminal/SSH远程连接工具等）来进行打印处理。

所以如果在Linux下的Terminal上打印`str1`和`str2`的话，应该就是`str1`正常打印，`str2`出现乱码了，因为Terminal的编码一般是UTF-8的。

那如果想要实现**多平台下表现一致**的特性的话，比如说保证`class`的doc在多平台下打印出的效果一致或者写入效果一致（不会乱码）的话，我们该怎么做？下面引出了本篇内容的重点。

### 3. Unicode，Python的好伙伴儿
一门热门起来的编程语言，首先要满足的需求之一就是，对不同国家和地区所用字符的友好支持。Python很好的做到了这一点，至少，在它正式引入Unicode编码以后是这样。

我们都知道Python是一门世界通用的编程语言，如果它的源代码文件中出现的都是ASCII支持的字符，那Python会以ASCII编码的格式处理程序。不过，一旦源代码中出现了ASCII不支持的字符，它该怎么办？我用一张图来回答你吧

![](http://lyfing.qiniudn.com/blog/2014-01-09/unicode-in-python.png )
(图片来源：http://nltk.googlecode.com/svn/trunk/doc/book/ch03.html)

**也就是说，所有超出ASCII范围的字符的处理工作，无论在输入之前，或者输出之后是什么编码格式的，它们在Python的执行内存中，都被统一转换(decode)为Unicode格式来进行程序处理。**

还是上节末尾那个例子，如何保证`str1`和`str2`在各个平台下都能正常打印呢，这里的办法是：将`str1`和`str2`转换成`Unicode`类型。

#### 3.1 什么是`Unicode`类型？那`str1`和`str2`又是什么类型的呢？

我们先看看`str1``str2`的类型

    >>> print type(str1), type(str2)
    <type 'str'> <type 'str'>

从上面可以看出，`str1、2`是`str`类型的，也就是通常意义下的字符串类型。

`Unicode`类型则是Python内建的一种用来“一统江湖”的字符类型(type)。 

既然已经有了用来容纳字符的类型——`str`——为何还要添加一种`Unicode`字符类型呢？要回答这个问题，我们先来想想如下几个现实中可能会遇到的情况：

 - 在某个class的`doc`中，需要同时用到多个国家的字符和符号来描述class的功用。试想如果此时再用单一的、相对狭隘的编码配合`str`来表述，可能出现不可预期的错误。这是因为字符所处的字符集和编码不一致，同一串16进制码(code points)在不同的字符集中可能表示不同的字符，更何况，同一个源代码文件中只可以定义一种编码格式

 - xml文件的解析(parse)问题，xml文件中可能包含不同国家和区域的字符和符号，此时如果使用仅适用于某个区域的字符编码，可能出现无法识别某些字符而导致甚至抛异常或程序错误等等糟糕情况。

于是，一统江湖、或者说差点就一统江湖的[Unicode编码][unicode]出现了。再接着，Python为了更加友好地支持各种国家/区域的字符，内建了新的字符容器`Unicode`。于是上面两个问题以及近似的一类问题都可以迎刃而解了。

#### 3.2 Unicode在Python中的使用

`Unicode`型的字符容器，在Python的具体应用中有着诸多优势。下面列举几个。

例子：在源代码中定义`Unicode`类型。  
具体步骤，创建`test.py`文件，在其中添加这样的内容

    # coding:utf-8
    def hello():
        str1 = u'我是大坏蛋'
        str2 = '你是小淫魔' 
        return (str1, str2)

接着在命令行`cd`至`test.py`所在目录，键入`python`进入Python Shell，在Shell中输入以下内容：

    >>> import test
    >>> str1, str2 = test.hello()
    >>> str1
    u'\u6211\u662f\u5927\u574f\u86cb'
    >>> str2
    '\xe6\x88\x91\xe6\x98\xaf\xe5\xb0\x8f\xe6\xb7\xab\xe9\xad\x94'
    >>> print str1
    我是大坏蛋
    >>> print str2
    鎴戞槸灏忔帆榄

我们看到在`hello()`定义时，`str1`是`Unicode`类型的，而`str2`则是普通的'str'字符型。也许在这里还没看出`Unicode`类型的优势。但是当需要打印变量内容时，二者的优劣便体现出来了：`Unicode`类型的`str1`无需进行特别编码便能正常打印出中文字符，但`str`类型的`str2`却出现了乱码情况。

此处可以介绍`Unicode`类型的很多优点了：
    
 - 声明简单 只需要在原字符前添加该类型标记`u`即可，例如`s1 = u'小伙伴儿'`
 - 代码编译期间，自动从声明的原始编码转码至Unicode，并创建相应的`Unicode`类型
 - 在需要打印(如`print`）或者输入输出（如`read`或`write`）时，`Unicode`类型用着尤其顺手。因为你只需要像操作ASCII一样打印和输入输出就行了，而无需关心编码、乱码问题。这是因为Python会帮`Unicode`类型的字符做编码转码工作，而这也正是`Unicode`类型的字符不会出现乱码的原因。Python工作内容如下：

     - 当需要输入中文字符时，Python会首先调取字符输入程序（命令行或者read函数）的编码格式，然后将输入的字符以该编码格式进行相应转换，即把原字符转码成`Unicode`类型的字符。

         - 例如在Windows中的命令行下使用的Python Shell，其编码格式是`GBK`，当你在shell中键入`string1 = u'我是李寻欢'`时，会发生如下事件：
         - 1, Python获取当前命令行编码（`sys.stdin.encoding`），为cp936(即GBK)
         - 2, 将输入的汉字`我是李寻欢`按GBK编码转码成Unicode的形式，并据此创建`Unicode`类型的字符串，赋值给`string1`

     - 当需要打印输出时，Python会首先调取字符输出程序（命令行或者输出函数）的编码格式，然后将该字符串编码成字符输出程序所用的编码（这样字符输出程序就不会因为认不出编码而出现乱码），接着字符输出程序将编码后的字符输出到目的地。该处理过程是字符输入程序（上一条）的逆过程，此处不再详细介绍这一过程

#### 3.3 如何将`str`type转换成`Unicode`type

这是本节最开始提出的那个问题：如何将`str`类型的字符串`str1`和`str2`转换成`Unicode`类型？
这也是下节要谈论的编码与解码的问题。

### 4. 编码（encode）与解码（decode）

如何将`str`类型的字符串`str1`和`str2`转换成`Unicode`类型？以及上一节最末提到关于的字符编码在GBK和Unicode之间的转换，涉及到的都是编码与解码的问题。统一来说就是

从`Unicode`类型到GBK或UTF-8等编码的转换，叫做**编码(encode)**；而这一过程的逆过程，则叫做**解码(decode)**。

一开始我老是搞不清楚到底从哪到哪是编码，从哪到哪是解码，后来用多了自然也就清楚了。相信初次接触这个概念的人也会有和我一样的困惑，下面用一张图来说明一下编码和解码的方向：

![](http://lyfing.qiniudn.com/blog/2014-01-09/encode-decode.jpg )

关于解码和编码的实现方法，网上已有很多，这里不再重述。下面引用一篇博客中的内容来做个总结：

> 字符串在Python内部的表示是Unicode编码。因此在做编码转换时，通常需要以Unicode作为中间编码，即先将其他编码的字符串解码(decode)成Unicode，再从Unicode编码(encode)成另一种编码。  
 `decode`的作用是将其他编码的字符串转换成Unicode编码，如`str1.decode('gb2312')`，表示将gb2312编码的字符串`str1`转换成Unicode编码；  
 `encode`的作用是将Unicode编码转换成其他编码的字符串，如`str2.encode('gb2312')`，表示将Unicode编码的字符串`str2`转换成gb2312编码   因此，转码的时候一定要先搞明白，字符串str是什么编码，然后decode成Unicode，然后再encode成其他编码。（[原文链接][src]）

遇到乱码问题，按上述思路分析一遍，再将乱码所在字符串按它的原始编码`decode`成Unicode类型的，再使用Unicode类型的编码进行输入、输出即可解决乱码问题。

ref:

 - [http://docs.python.org/2/howto/unicode.html](http://docs.python.org/2/howto/unicode.html )

[supported-encoding]:http://docs.python.org/2/library/codecs.html#standard-encodings
[pep-0263]:http://www.python.org/peps/pep-0263.html
[unicode]:http://zh.wikipedia.org/wiki/Unicode
[src]:http://blog.csdn.net/lxdcyh/article/details/4018054
