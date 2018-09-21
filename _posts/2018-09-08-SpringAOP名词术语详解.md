---
layout: post
title: Spring AOP名词术语详解
---

# 概述
SpringAOP的术语(其实是Java AOP)并不形象和易于理解，有时候比较容易忘记和混淆概念。再者其实AOP中的术语也有重要和次重要之分，一股脑全盘理解有点混沌，所以本篇介绍将以优先掌握重要术语为主，理解次重要术语为辅，这样一来比较容易记住这些概念。

面向切面编程中所谓的切面，其实可以看成是水平切入(假设代码执行方向为自上而下)代码流程中一个平面，通常用一个普通类来实现。

先说一下切面在整体运行过程中。重度依赖的的以下三个关键信息：
 - ① 在正常程序执行的哪个点切入(由@Pointcut声明)；
 - ② 切入后要执行的动作(@Advice)；
 - ③ 动作@Advice在切点的哪个阶段执行(@Before/After/Around等) 


# 名词细讲

## 重要术语

### @Aspect (重要度:🥇)

也就是切面，它通常需要包含上述所有三个关键信息（虽然总有人说它只包含①和②）

### @Pointcut (重要度:🥇)

Pointcut声明了代码的特征信息（也就是期望切入切点的位置信息），以便AOP程序在执行切入时首先匹配当前代码是否符合特征，继而决定是否要在此切入切面。

由于SpringAOP只支持方法method级别的切入，所以，Spring中的Pointcut只能声明方法的特征信息。下面举例说明Pointcut是如何声明方法的特征信息的

```
// 如下Pointcut表示：拦截com.xyz.service包下的每个类的所有方法（未包含sub-package中的类方法）
<aop:pointcut id="xxx" expression="execution(* com.xyz.service.*.*(..))" />

// 如下Pointcut表示：拦截项目中的所有public方法
<aop:pointcut id="xxx" expression="execution(public * *(..))" />
```

除了使用`expression`声明目标方法的特征之外，还可以使用`within`、`target`、`args`等关键字，Pointcut更多详情请查阅：https://docs.spring.io/spring/docs/2.0.x/reference/aop.html "6.2.3. Declaring a pointcut" 一节

### @Advice（重要度:🥇）
Advice：切面实际要执行的动作，它通常就是切面所在类中的一个方法(method)。它通常还包含了在被代理方法的哪个阶段执行的信息，具体的阶段有：
 - `Before`：在被代理方法执行之前执行，它不能控制被代理方法的执行与否
 - `After returning`：在被代理方法正常return之后执行
 - `After throwing`：在被代理方法抛出异常后执行
 - `After (finally)`：在上述两种情况（正常return或抛出异常）之后执行
 - `Around`：在被代理方法前后执行。只有该方法才能控制被代理方法执行与否（）


### @Joinpoint（重要度:🥇）
Joinpoint：它通常作为@Advice方法的唯一入参，用来表示被切入切面处的方法（被代理方法）并控制其运行，它实际是控制被代理方法执行的句柄。

例如对于有返回值的@Advice方法，你需要主动通过`return JoinPoint.proceed()`才可以得到被代理方法的原始返回值，如果直接`return 其他值`并且不调用`JoinPoint.proceed()`，那么被代理方法将直接被忽略不执行。

------

## 次重要术语

### Weaving（重要度:🥈）
将切面@Aspect应用于被代理方法所属类/接口并生成Advised object对象(即下方的@Target object)的过程。该过程可在三个不同的阶段被独立实现：编译期、加载期和运行时。Spring的实现为运行时生成。

### @Target object（重要度:🥈）
也称`Advised object`. 被代理方法所属的对象。由于SpringAOP的实现是运行时代理，所以@Target object是对实际类或接口做了封装的代理对象。

比如，对`AccountDAO`接口中的每个方法做切面切入时，在@Advice方法体中获取到的被拦截的对象名称可能为：`AccountDAO$CGLIB`(即AccountDAO的代理对象)，而不是`AccountDAO`.

### @Introduction（重要度:🥉）
为@Target object动态新增接口或方法的过程。一般较少用的
