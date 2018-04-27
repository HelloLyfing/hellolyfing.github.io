---
layout: post
title: PHP对象stdClass如何判断是否为空
tags: [Tech]
---

## 背景
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

至此我们发现，通过将对象转为array以便间接判断stdClass是否为空是更高效的做法

## 进阶
当然，上面两种判断方法都是在：stdClass确实为空对象这一前提下进行的，那么如果stdClass是一个具有200个类属性的方法时，二者执行效率孰优孰劣？

我们先来创建一个有200个类属性的对象
```
$obj = new stdClass();

for ($i = 0; $i < 200; $i++) {
    $attr = "{$i}rand{$i}";
    $obj->$attr = $i;
}
```

再分别使用方式一和方式二对这个对象进行判空操作，对这两种方法分别执行500万次后发现：
 - 方法一（转为array）执行500万次耗时仍然维持在0.8s左右；
 - 方法二在执行同样次数后，耗时比之前增加了5s，达到了23.45s

## 结论
通过将对象转为array以间接判断stdClass是否为空，是更高效的做法