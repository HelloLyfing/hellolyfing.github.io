---
layout: post
title: "MySQL技能二三事"
comments: true
tags: [Tech, Database, Mysql]
description: "MySQL技能二三事"
---

# 其一: int(10)意欲何为

MySQL的Column-Type中的**int**一类，不像其他Type如**char**、**varchar**是根据最大size来预置存储空间的，**int**这一类的Column-Type所指定的size是作为显示宽度来存储的(涉及到size不足前端补零的场景-zerofill），它们所占用的实际空间大小是[固定][int-type-doc]的。所以想通过`int(10)`来节省存储空间的想法是无效的。

# 其二: index的prefix length

最近的一个项目需要存储互联网上的URL，并生成唯一ID，所以数据表里边会有`urlcmpr`一栏

 > urlcmpr: 适当压缩过的url，用0、1代表http[s]，62进制的code代表hostname。示例：0h^/Mucinex-Sinus-Max-Pressure-Caplets-Count/dp/B009XQHSUC/ref-id=379

<!--more-->

当时的建表语句是这样的（其实项目用的是SQLAlchemy的ORM-Model，这里只是复现一下当时的情况）

```
CREATE TABLE `urls` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `urlcmpr` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_urlcmpr` (`urlcmpr`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

查看建好的表信息

```
mysql> DESC urls;
+---------+--------------+------+-----+---------+----------------+
| Field   | Type         | Null | Key | Default | Extra          |
+---------+--------------+------+-----+---------+----------------+
| id      | int(11)      | NO   | PRI | NULL    | auto_increment |
| urlcmpr | varchar(255) | YES  | UNI | NULL    |                |
+---------+--------------+------+-----+---------+----------------+
```

首先插入两条记录：

```
insert into urls (urlcmpr) values 
  ('0h^/Mucinex-Sinus-Max-Pressure-Caplets-Count/dp/B009XQHSUC/ref-id=0001'),
  ('0h^/Mucinex-Sinus-Max-Pressure-Caplets-Count/dp/B009XQHSUC/ref-id=0002');
```

因为我们已经为`urlcmpr`建好了唯一索引，我们来试试重复插入一下：

```
insert into urls (urlcmpr) values 
  ('0h^/Mucinex-Sinus-Max-Pressure-Caplets-Count/dp/B009XQHSUC/ref-id=0001');
```

不出所料，报错了

```
ERROR 1062 (23000): 
Duplicate entry '0h^/Mucinex-Sinus-Max-Pressure-Caplets-Count/dp/B009XQHSUC/ref-i'
for key 'idx_urlcmpr'
```

这个原本也没什么，但由于下方已删除的那部分描述的缘由，我开始关注错误报告所反映的一个信息：`urlcmpr`栏是255字符的，但据此建立的`unique-key`却只有64字符的长度，这是怎么回事？

其实后来才发现这个只是显示了*唯一Key*的前64个字符而已，其实*唯一Key*并不是只有64个字符。

<s>这里建唯一索引的目的是为了在重复误插入的时候被拒绝并报错，结果在程序准备正常插入的时候，MySQL也开始拒绝并报错了。但是事后当我试图重现这个问题时却发现，本段所描述的问题是不存在的！</s>

总之由于上面的一些误解（经过一步步查资料又自己推翻）就开始查资料了，查着查着找到了[官方文档][index-prefixes-doc]里的这样一段话（略有删减）：

 > Indexes can be created that use only the leading part of column values, using col_name(length) syntax to specify an **index prefix** length. Prefix lengths are given in characters... That is, index entries consist of the first length characters of each column value for CHAR, VARCHAR, and TEXT columns, and the first length bytes of each column value for BINARY, VARBINARY, and BLOB columns... using column prefixes for indexes can make the index file much smaller, which could save a lot of disk space and might also speed up INSERT operations.

基本就是在说：chars类的column，在建立索引的时候会有**index prefix length**一说，也就是只取本栏开头的若干个字符做索引（而不是本栏全部内容），这样做的好处是可以节省存储空间。

<s>虽然官方文档中没有说到为什么当一栏内容过大而且你没有设定它的index-prefix-length时，它的这个值会被设为64，但目前来看事实似乎就是这样的。</s>

如果是存储较长字符串(比如255 chars)的唯一索引，它的`prefix-length`该怎么定制（从节省存储空间的角度看）？用上面的例子做示例：  
`UNIQUE KEY 'idx_urlcmpr' ('urlcmpr'(100))`  
即在想要建立索引的栏名后面的括号中指定prefix-length。

我们再来试试看

```
CREATE TABLE `urls2` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `urlcmpr` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_urlcmpr` (`urlcmpr`(100))
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

插入一段字符个数为100的string:   

```
0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZab

Insert OK
```

在上一段string尾部加一个字符`c`(101个字符长度)再试试：  

```
0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabc 
ERROR 1062 (23000): Duplicate entry
```

无法插入了。上面的两步说明我们设定的唯一索引的`prefix-length`成功了，不过它也反映出一个问题：如果你要存储的内容在前半部分重复度很高，那么你的唯一索引可能会因为`prefix-length`过小而导致无法插入(如果两个string在开头的`prefix-length`以内是相同的)。因为确实已经遇到无法插入的问题了(500万+的URL），考虑了一下后我把*UniqueKey*的限制给去掉了，改用普通Key: `KEY 'idx_urlcmpr' ('urlcmpr'(100))`，是否重复则交由程序来判断，一来可以节省存储空间（Key也是要被存储记录的，不宜太长），二来防止将来有高度相似的URL无法插入。

# 其三：MySQL搜索校验时的Case -Sensitive问题

由于业务需要，在存储这么多url的时候，每存储一条，还需要为它生成一个唯一码(unique code)。网上已经有不少优秀的唯一ID生成器了，但想要并入业务中还有些麻烦，再者业务要求很简单，保证code在单亿数量级以内唯一，于是自己就顺手写了一个62进制的[编码程序][my-62hex-link]来生成unique code。由于code生成器是62进制的，如果是10个字符长的话，可表示的组合便是62的10次方，换算成10进制的话应该是一个近乎天文的数字了，唯一性是绝对保证了；程序还带了code+1功能，方便用来自增，有兴趣或有需要的朋友可以去看一下。

言归正传，利用上面的code生成器产出的code是大小写敏感的，即`fooBar`、`Foobar`和`fOOBaR`分别代表着不同的10进制数值。新建一张表，把它们三个存进MySQL中：

```
# 新建表
CREATE TABLE `uni_code` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `code` varchar(10) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
Query OK, 0 rows affected (0.33 sec)

# 插入三条数据
INSERT INTO uni_code (code) VALUES ('foobar'),('fooBar'),('fOOBaR');
Query OK, 3 rows affected (0.03 sec)

# 查看刚刚插入的数据
mysql> SELECT * FROM uni_code;
+----+--------+
| id | code   |
+----+--------+
|  1 | foobar |
|  2 | fooBar |
|  3 | fOOBaR |
+----+--------+
3 rows in set (0.00 sec)
```

我们再试着取出其中一条数据来：

```
mysql> SELECT * FROM uni_code WHERE code = 'foobar';
+----+--------+
| id | code   |
+----+--------+
|  1 | fooBar |
|  2 | fooBar |
|  3 | fOOBaR |
+----+--------+
3 rows in set (0.03 sec)
```

预期中的**坑**出现了。我的code是大小写敏感的，但MySQL在查询校验的时候似乎忽略了这一点，直接把三条数据全部取出来了！

当然你可以通过手动添加`Binary`[操作符][binary-operator-doc]将字符转换成字节来进行查询校验，或者指定[collation][charset-general-doc]规则来进行查询校验

```
mysql> SELECT * FROM uni_code WHERE BINARY code = 'foobar';
+----+--------+
| id | code   |
+----+--------+
|  1 | foobar |
+----+--------+
1 row in set (0.00 sec)

mysql> SELECT * FROM uni_code WHERE code COLLATE utf8_bin = 'foobar';
+----+--------+
| id | code   |
+----+--------+
|  1 | foobar |
+----+--------+
1 row in set (0.09 sec)
```

但一来我用的ORM，使用原生SQL (关键字`Binary`、`Collate`) 与使用ORM目的背离；二来四处滥用这些关键字会让程序代码混乱。最好能让MySQL在数据库层就完成大小写敏感的查询校验。

查了查资料发现MySQL在查询校验的时候所使用的`Collate`规则集是有默认值的，正是这个规则集决定了字符比对的结果。那么`Collate`是什么呢？

在上面给出的[官方文档][charset-general-doc]已经很好的解释了`Collate`的作用，我在这里再赘述一二：

数据库中有字符集**charset**和规则集**collate**一说，字符集自不必多说。规则集是**在给定字符集上进行字符对比的规则**，比如大小写敏感否 (CI or CS)、`äöü`与`aou`对等否等规则，每个字符集都有默认规则集。

上面的`... WHERE code COLLATE utf8_bin = ...`就是指定了字符对比的规则集。这个规则集是大小写敏感的，它会覆盖默认的规则集。是的，MySQL中的每一个单元(Column、Table、Database)都可以有自己的字符集和规则集，我们通常只是指定字符集，MySQL则帮我们指定了该字符集对应的默认[规则集][show-charset-doc]。

这么一来就好办了，看来是MySQL为我指定了utf8编码下的默认规则集，而这个规则集在字符比对时是CI的。只要我们修改一下表(或者栏)的规则集，让它支持大小写敏感的比对就好了。

我们先来看看`utf8`的默认规则集，及可选规则集(下面内容可能会因版本、系统而异)：

```
mysql> show CHARACTER SET WHERE charset LIKE "utf8";
+---------+---------------+-------------------+--------+
| Charset | Description   | Default collation | Maxlen |
+---------+---------------+-------------------+--------+
| utf8    | UTF-8 Unicode | utf8_general_ci   |      3 |
+---------+---------------+-------------------+--------+
1 row in set (0.00 sec)
```

从上可以看出，utf8编码的默认规则集是`utf8_general_ci` (规则集的命名规则通常是"字符集_规则集_大小写敏感否"的格式)，可以看到最后的`_ci`表名了这个规则集是大小写不敏感的。我们找一个支持大小写敏感比对的规则集出来：

```
mysql> SHOW COLLATION WHERE Charset LIKE 'utf8';
+--------------------------+---------+-----+---------+----------+---------+
| Collation                | Charset | Id  | Default | Compiled | Sortlen |
+--------------------------+---------+-----+---------+----------+---------+
| utf8_general_ci          | utf8    |  33 | Yes     | Yes      |       1 |
| utf8_bin                 | utf8    |  83 |         | Yes      |       1 |
| utf8_unicode_ci          | utf8    | 192 |         | Yes      |       8 |
| utf8_icelandic_ci        | utf8    | 193 |         | Yes      |       8 |
| utf8_latvian_ci          | utf8    | 194 |         | Yes      |       8 |
| utf8_romanian_ci         | utf8    | 195 |         | Yes      |       8 |
| utf8_slovenian_ci        | utf8    | 196 |         | Yes      |       8 |
| utf8_polish_ci           | utf8    | 197 |         | Yes      |       8 |
| utf8_estonian_ci         | utf8    | 198 |         | Yes      |       8 |
| utf8_spanish_ci          | utf8    | 199 |         | Yes      |       8 |
| utf8_swedish_ci          | utf8    | 200 |         | Yes      |       8 |
| utf8_turkish_ci          | utf8    | 201 |         | Yes      |       8 |
| utf8_czech_ci            | utf8    | 202 |         | Yes      |       8 |
| utf8_danish_ci           | utf8    | 203 |         | Yes      |       8 |
| utf8_lithuanian_ci       | utf8    | 204 |         | Yes      |       8 |
| utf8_slovak_ci           | utf8    | 205 |         | Yes      |       8 |
| utf8_spanish2_ci         | utf8    | 206 |         | Yes      |       8 |
| utf8_roman_ci            | utf8    | 207 |         | Yes      |       8 |
| utf8_persian_ci          | utf8    | 208 |         | Yes      |       8 |
| utf8_esperanto_ci        | utf8    | 209 |         | Yes      |       8 |
| utf8_hungarian_ci        | utf8    | 210 |         | Yes      |       8 |
| utf8_sinhala_ci          | utf8    | 211 |         | Yes      |       8 |
| utf8_german2_ci          | utf8    | 212 |         | Yes      |       8 |
| utf8_croatian_ci         | utf8    | 213 |         | Yes      |       8 |
| utf8_unicode_520_ci      | utf8    | 214 |         | Yes      |       8 |
| utf8_vietnamese_ci       | utf8    | 215 |         | Yes      |       8 |
| utf8_general_mysql500_ci | utf8    | 223 |         | Yes      |       1 |
+--------------------------+---------+-----+---------+----------+---------+
27 rows in set (0.00 sec)
```

纳尼... 27个选择几乎全是大小写不敏感的（可见大小写敏感的用例不是很广），幸好还有个`utf8_bin`，它是支持大小写敏感比对的(用字节而非字符进行比对)。

我们把上面的uni_code table的`code`一栏的`collate`换成`utf8_bin`：

```
mysql> ALTER TABLE uni_code MODIFY code
    ->   VARCHAR(10)
    ->   CHARACTER SET utf8
    ->   COLLATE utf8_bin;
Query OK, 3 rows affected (1.44 sec)
Records: 3  Duplicates: 0  Warnings: 0

SHOW CREATE TABLE uni_code;
CREATE TABLE `uni_code` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `code` varchar(10) CHARACTER SET utf8 COLLATE utf8_bin DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8
```

我们看到`code`一栏的**collate**属性已经改过来了。Table和DB的改动格式见[官方文档][set-charset-doc]。

再说说线上改表的问题。上面那张表也就3行数据，改动一栏尚且花了1.44s的时间，这要在500万+的数据表上做改动得花去多长时间，假如还要考虑线上表的访问、新增、更新请求就更需谨慎了。这里推荐一个[在线改表工具][alter-t-online]，好用的很。当时是暂停服务进行的改动整张表的collate属性的操作，有两栏是varchar的(size分别是10、255)，500万+的数据十几分钟就搞定。

# 其四: Update语句并不一定真的会"Update"


有一张如下的表：
```
mysql> describe big_msg;
+-------+-------------+------+-----+---------+-------+
| Field | Type        | Null | Key | Default | Extra |
+-------+-------------+------+-----+---------+-------+
| key   | varchar(20) | YES  | MUL | NULL    |       |
| val   | text        | YES  |     | NULL    |       |
+-------+-------------+------+-----+---------+-------+
2 rows in set (0.01 sec)
```

插入一行：
```
mysql> INSERT INTO big_msg (`key`, `val`) VALUES ("test-key", "test-val");
Query OK, 1 row affected (0.00 sec)
```

查看：
```
mysql> UPDATE big_msg SET `val`="test-val" WHERE `key`="test-key";
Query OK, 0 rows affected (0.00 sec)
Rows matched: 1  Changed: 0  Warnings: 0
```

注意这里的`0 rows affected (0.00 sec)`，也就是说这里并没有更新本行。这个在普通场景下并无大碍，不过当你使用的类似`PHP PDO`面向对象的数据库类库，而且你的`Statement`又依赖于`row affected`这个返回值时（例如PHP-PDO的`PDOStatement::rowCount ( void )`），这里就需要尤其注意一下了，**如果你的更新值和原有值相同，则本次更新会被忽略, 返回值`affected rows`将为0**。看官方文档中的说明：

> If you set a column to the value it currently has, MySQL notices this and does not update it.


[int-type-doc]: http://dev.mysql.com/doc/refman/5.1/en/integer-types.html
[index-prefixes-doc]: http://dev.mysql.com/doc/refman/5.1/en/create-index.html
[my-62hex-link]: https://github.com/HelloLyfing/tiny-works/blob/master/HexConverter/HexCvter.py
[binary-operator-doc]: http://dev.mysql.com/doc/refman/5.5/en/charset-binary-op.html
[charset-discuss]: http://dba.stackexchange.com/a/15254/53044
[charset-general-doc]: http://dev.mysql.com/doc/refman/5.5/en/charset-general.html
[show-charset-doc]: http://dev.mysql.com/doc/refman/5.5/en/charset-mysql.html
[set-charset-doc]: http://dev.mysql.com/doc/refman/5.5/en/charset-table.html
[alter-t-online]: http://openarkkit.googlecode.com/svn/trunk/openarkkit/doc/html/oak-online-alter-table.html
