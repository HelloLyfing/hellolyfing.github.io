---
layout: post
title: Spring同一切点多个AOP Advice导致错用数据库的问题排查
---

# 一、背景描述
#### 背景1
环球捕手Java后端项目，对Mysql的访问由Spring + Mybatis实现。开发人员对Mysql的访问实行了读、写分离。对于DAO层的数据增、删和更新等写入的请求，需要走master主库；对于数据读取请求，则需要走slave从库。开发人员通过SpringAOP切面的方式来设置每个DAO接口的方法级别的数据源，即设置每个方法是访问master还是slave。

其实现方式为：DAO层接口所有方法的数据源，默认走主库，但可以通过为方法添加`DataSource`注解的方式，来主动设置接口内该方法的数据源。

#### 背景2
位于业务下层的中台服务化 - 用户服务化(User-Center-Service)，需要对环球捕手后端项目中，所有直接通过DAO接口访问用户数据表的操作，替换为调用用户服务化的RPC接口。

由于业务方日常需求过多，无法抽出人力帮助我们梳理接口和改动接口，我们思考后设计出：无需改动任何业务代码，直接通过拦截DAO层接口访问，并替换为我方RPC服务的方式，来实现对环球捕手项目中所有对用户数据表直接访问的接口的收口。

#### 出现问题
背景1的需求已经上线并稳定运行了很长时间。在此基础上，我们针对背景2开发出了小范围接口替换的demo版本，写好测试用例并测试没有问题后，我们将这个demo版本发布上线。

但上线之后问题开始出现了：一些原本应该访问master主库的写操作，却访问了slave从库，然而从库是禁止写操作执行的，于是写操作失败并抛出了如下异常：

```
### Error updating database.  Cause: java.sql.SQLException: 
The MySQL server is running with the --read-only option so it cannot execute this statement.
```

# 二、问题分析

### 2.1 技术实现细节
在分析 *数据源为什么会连错的情况* 之前，我们最好先了解一下上述每个背景的技术实现的细节。

**先说背景1的技术实现细节**。如果有多个数据源，如何通过AOP实现数据源的动态切换呢？这个方案本身已经非常成熟了，这里不再赘述。总结来说就是：
 1. 首先给DAO中需要切换数据源的方法(就拿`selectAccountById`方法举例吧)添加自定义注解，注解内容包含了数据源信息。也就是为方法打上一个自定义标签
 1. 编写切面Advice，在方法`selectAccountById`执行前，解析该方法的自定义标签，判断该方法期望使用的数据源，并将**数据源信息暂存起来**
 1. 新建类，并继承Spring JDBC的`AbstractRoutingDataSource`类，覆盖其`determineCurrentLookupKey`方法。`determineCurrentLookupKey`用于Spring在获得数据库连接(getConnection)之前执行，以便在出现多数据源的情况时，由该方法确定使用哪个数据源key。我们覆盖该方法，并将上一步**暂存**的**数据源信息**作为返回值返回

由于DAO层每个方法在执行之前，都会调用一次`determineCurrentLookupKey`以获取该方法需要的数据源(这句话不够严谨，后面细讲)。这样一来，同一DAO中的每个方法，便都可以指定数据源了，而背景1需要实现的需求也就迎刃而解。

背景1技术结构图如下：

![](/images/2018-09/dao-aop-设数据源切面.png)


**背景1示例代码**

为DAO中的方法添加自定义的@DataSource注解的示例代码如下：

```
@Repository
public interface AccountDao {
    /**
     * 接口未设置数据源，则默认走master主库，效果等同于添加注解: @DataSource("master")
     */
    int insertAccount(AccountDO accountDO);

    /**
     * 设置该方法的数据源为slave从库
     */
    @DataSource("slave")
    AccountDO selectAccountById(int accountId);
}
```

AOP中的Advice（执行动作）的示例代码如下。它只做一件事：解析每个方法上的数据源注解，如果存在则交由`DataSourceKeyHolder`类暂存。

```
public class MultipleDataSourceAspectAdvice {
    
    // 切点切入时机为: @Before
    public void invoke(JoinPoint joinPoint) {
            Method method = joinPoint.getCalledMethod();
            // 如果切点处的方法，含有@DataSource注解，则获取其注解值并暂存到DataSourceKeyHolder类中
            if (method != null && method.isAnnotationPresent(DataSource.class)) {
                DataSource dataSourceKey = method.getAnnotation(DataSource.class);
                DataSourceKeyHolder.setDataSourceKey(dataSourceKey);
            }
        }
    }

}
```

当然还有一个极为关键的类`DataSourceKeyHolder`，它的作用有二：1）使用ThreadLocal类型的类变量暂存上一步从方法上解析到的数据源key；2）继承`AbstractRoutingDataSource`类，并覆盖`determineCurrentLookupKey`方法，该方法将返回之前暂存的数据源key

```
public class DataSourceKeyHolder extends AbstractRoutingDataSource {

    private static ThreadLocal<String> KEY_HOLDER = new InheritableThreadLocal<>();

    public static void setDataSourceKey(String dataSource) {
        KEY_HOLDER.set(dataSource);
    }

    @Override
    protected Object determineCurrentLookupKey() {
        // Spring JDBC将执行本方法获取数据源key，以便选择数据源
        String dataSource = KEY_HOLDER.get();

        // 数据源被索取之后，立即重置KEY_HOLDER，防止造成后续数据源的污染
        KEY_HOLDER.set(null);

        return dataSource;
    }
}    
```

**在此需要拆解一下上述方案**。看上去它只有一个AOP切面，该切面完成了：解析DAO的方法注解并暂存方法注解中的数据源信息的工作。

需要注意的是：**上述方案其实还有另外一个AOP切面，并没有被提到！**这另外一个切面就是：Mybatis对DAO接口解析、匹配对应mapper文件中的SQL，并最终通过Spring JDBC进行数据库查询的切面，我们暂且称其为MybatisMapper切面吧。正是这个切面，间接调用了`determineCurrentLookupKey()`方法，从类线程变量`KEY_HOLDER`中获取暂存的数据源信息，并完成`KEY_HOLDER`的重置工作。

所谓间接调用是指，MybatisMapper调用了Spring JDBC，而后者又调用了`determineCurrentLookupKey()`方法。



**再说背景2的技术实现细节**

背景2的需求：把DAO中直接访问数据库的方法，动态替换为调用远程RPC服务，避免DAO的方法继续直接访问数据库。当然在此过程中要保证替换前后方法的返回值一模一样。

背景2需求的实现，分为两个部分：1）给需要替换为RPC调用的DAO方法添加自定义注解；2）**在AOP中拦截该DAO方法的执行，识别该自定义注解，并转为调用远程RPC服务，将RPC的返回结果格式化为该方法的返回类型，直接return结果，*并放弃对该方法的继续执行*。**

**背景2示例代码**

DAO中，同时结合了数据源的注解@DataSource，和调用RPC服务的注解@UseUserCenterApi。代码示例如下：

```
@Repository
public interface AccountDao {
    /**
     * 接口未设置数据源，则默认走master主库，效果等同于添加注解: @DataSource("master")
     */
    int insertAccount(AccountDO accountDO);

    /**
     * @DataSource设置该方法的数据源为slave从库
     * @UseUserCenterApi 标识该方法需要使用RPC返回结果，并放弃对数据库的访问
     */
    @DataSource("slave")
    @UseUserCenterApi
    AccountDO selectAccountById(int accountId);
}
```

借助AOP实现替换原方法调用的示例代码如下：

```
public class Switch2UserServiceApiInterceptor implements MethodInterceptor {

    // 切面切入时机为@Around
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        if (invocation.getMethod().isAnnotationPresent(UseUserCenterApi.class)) {
            // !!! 被拦截的方法不再正常向下执行，而且，后续其他切面的执行也被放弃
            return callUserCenterApi(invocation.getMethod(), invocation.getArguments());
        }

        // 在@Around型切入中，只有执行如下代码，被拦截的方法才会正常向下执行。
        // 此时如果有后续切面，只有调用该方法才能保证后续切面被执行
        return invocation.proceed();
    }

    // 远程调用实现细节
    private Object callUserCenterApi(arg1, arg2)...
}
```

### 2.2 AOP之间的相互影响
通过上面对技术实现细节的讨论，我们可以得出这样的结论：
 > 同一个接口（例如`selectAccountById`）最多可能会被三个切面作用。这三个切面按执行顺序排列分别是
  1. 第一个，背景1中负责解析数据源信息并暂存数据源key的切面 —— "数据源切面"
  1. 第二个，背景2中负责替换DAO对数据库的直接访问的 —— "替换RPC切面"
  1. 第三个，Spring中负责解析DAO接口并执行数据库查询的 —— "Mybatis切面"

![](/images/2018-09/dao-aop正常流程.png)

通过分析技术细节、切面执行顺序以及流程，有些人已经隐约能猜出本文标题的问题是怎么造成的了。

 - DAO层方法被执行前，假如“数据源切面1”设置（暂存）了数据源：
     - 按正常执行流程，“Mybatis切面3”将读取并重置该数据源信息，则下次其他方法被执行前，数据源仍然为默认数据源，不会造成问题和数据源污染
     
 - 然而两个切面之间插入了另外一个切面“替换RPC切面2”，该切面可通过直接返回RPC结果，放弃调用`joinPoint.proceed()`而导致后续切面不再被执行！这也就造成第一步“数据源切面1”暂存的数据源key，不会被清空，当任意DAO层其他方法被调用时，假如该方法未指定数据源，则会沿用上一次设置的数据源。这与“方法未指定数据源，则默认使用主库”的原始意图相左了。

![](/images/2018-09/dao-aop跳过执行.png)

# 三、解决问题（第一次尝试）

知道了原理，解决问题的方案也就能够给出来了，当时给出的解决方案有两个：
 - 方案1：改变切面的执行顺序，让“数据源切面”和"Mybatis切面"紧密且连续地执行。由于"Mybatis切面"的执行order永远在最后一位，所以“数据源切面”必须放在倒数第2位执行。这种方式可以保证前者设置的数据源，一定会由后者索取并重置掉
 - 方案2：改进“数据源切面”的代码逻辑，无论DAO层方法有无注解数据源，“数据源切面”始终在第一行就重置数据源为空，后续代码逻辑不变。这样有数据源注解的依然可以在之后被设置，而无数据源注解的则一定会使用主库

上面两个方案中，我选了后者，稍微改动代码发布上线后，脸就被啪啪啪地打了 —— 本文标题中描述的问题仍在低频地出现。

# 四、继续分析问题...

当时工期太赶，没时间继续深究，而且因为是兄弟团队的项目，每次发布和回滚都要别人配合，连发两次都出问题，实在是“无颜见江东兄弟”的心情。还好这时候领导@俊登场了，说由他来看下原因，我则继续按工期开发，于是才有了后面的故事。

此处补充一句，如果当时按方案1来进行，则问题就会消失了，但也就没机会听到后面的故事了。

# 五、神龙终现身，从更高维度审视问题

其实还有另外一个看似毫不相关的切面，一直被我们所忽略。而恰恰正是这个切面的运行机制，加上之前已经分析出的部分问题的结论，才导致错用数据源的问题仍在继续。这个神秘而熟悉的切面就是：事务切面。

然而，这个切面并不在DAO层这么低的维度，它的活动范围位于DAO层之上的Service层。此处介绍一下这个事务切面的作用：
 - 对任何以insert、update、delete等关键字开头的位于指定Service包中的方法，通过`DataSourceTransactionManager`进行事务管理
 - 这个事务管理器的特点是：一个方法内，事务数据源的初始化工作只进行一次，即在事务开始时。而且其事务初始化工作，也同样依赖了Spring JDBC定义好的`determineCurrentLookupKey()`接口规范，也就间接依赖了“数据源切面”中设置（暂存）的数据源信息。

那么数据源错用问题，在方案2上线后仍然低频出现的现象也就得到了解释：
 > 方案2中，“数据源切面”在每个DAO的每个方法被执行前，都会在一开始就执行重置数据源的操作；然而，“事务切面”在更高层拦截Service方法的执行时，事务的数据源的初始化工作会提前进行，而且只进行一次。然而此时，DAO层任何方法根本没来得及执行，那么“数据源切面”想要完成的重置数据源的操作也就不会发生了。

![](/images/2018-09/dao-aop-全局图.png)

从上图的切面关系中可以看出，使用方案1，即：改变切面的执行顺序，让“数据源切面”和"Mybatis切面"紧密且连续地执行，可以解决本文遇到的问题。

此外，返回数据源信息的`determineCurrentLookupKey()`方法如果增加如下逻辑判断，应该也可以解决问题：
 - 如果当前Context位于事务中，则返回主库数据源信息





