---
layout: post
title: PHP对象stdClass如何判断是否为空
tags: [Tech]
---

先来看下普通的为空判断，应用于stdClass类是什么表现：

```
$obj = new stdClass();

if (!$obj) {
    echo '!(obj) = TRUE' . "\n";
} else {
    echo '!(obj) = FALSE' . "\n";
}

if (empty($obj)) {
    echo 'empty(obj) = TRUE' . "\n";
} else {
    echo 'empty(obj) = FALSE' . "\n";
}

if (is_null($obj)) {
    echo 'is_null(obj) = TRUE' . "\n";
} else {
    echo 'is_null(obj) = FALSE' . "\n";
}
```

输出结果（请先自己预测下？）：

```
!(obj) = FALSE

empty(obj) = FALSE

is_null(obj) = FALSE
```

不过在我接触过的各个语言中（Java、Python、Javascript），所有常规对象在进行布尔判断时都不为空，也算是符合业界通用的规则标准了

那么想要判断一个stdClass是否为空时，应该使用什么方法呢？
业界通用的做法是：将stcClass转换（cast）为数组，再对数组进行判空操作：

```
$obj = new stdClass();

$obj_arr = (array) $obj;

if (!$obj_arr) { 
    echo 'bool judge false' . "\n";
} else {
    echo 'bool judge true' . "\n";
}

```

当然还有人提出这样的想法：使用get_object_vars方法获取这个对象的类属性，通过判断其返回值来间接进行判空的操作

```
$obj = new stdClass();

$obj_attr_arr = get_object_vars($obj);

if (!$obj_attr_arr) {
    echo 'bool judge false' . "\n";
} else {
    echo 'bool judge true' . "\n";
}

```

那么这两种判空方式孰优孰劣呢？我对这两种方法分别执行500万次后发现
 - 方法一（转为array）执行500万次耗时仅为0.8s左右；
 - 方法二在执行同样次数后，耗时达到了惊人的17.8s左右

显然，通过将对象转为array以便间接判断stdClass是否为空是更高效的做法


