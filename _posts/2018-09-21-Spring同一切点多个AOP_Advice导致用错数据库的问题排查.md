---
layout: post
title: Spring同一切点多个AOP Advice导致用错数据库的问题排查
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

但上线之后问题开始出现了：一些原本应该访问master主库的写操作，却用错了访问方式，直接访问了slave从库，然而从库是禁止写操作执行的，于是写操作失败并抛出了如下异常：

```
### Error updating database.  Cause: java.sql.SQLException: 
The MySQL server is running with the --read-only option so it cannot execute this statement.
```

# 二、问题分析
在分析 *数据源为什么会连错的情况* 之前，我们最好先了解一下上述每个背景的技术实现的细节。

**先说背景1**。如果有多个数据源，如何通过AOP实现数据源的动态切换呢？这个方案本身已经非常成熟了，这里不再赘述。总结来说就是：
 1. 首先给DAO中需要切换数据源的方法(就拿`selectAccountById`方法举例吧)添加自定义注解，注解内容包含了数据源信息。也就是为方法打上一个自定义标签
 1. 编写切面Advice，在方法`selectAccountById`执行前，解析该方法的自定义标签，判断该方法期望使用的数据源，并将**数据源信息暂存起来**
 1. 新建类，并继承Spring JDBC的`AbstractRoutingDataSource`类，覆盖其`determineCurrentLookupKey`方法。`determineCurrentLookupKey`用于Spring在获得数据库连接(getConnection)之前执行，以便在出现多数据源的情况时，由该方法确定使用哪个数据源key。我们覆盖该方法，并将上一步**暂存**的**数据源信息**作为返回值返回

由于DAO层每个方法在执行之前，都会调用一次`determineCurrentLookupKey`以获取该方法需要的数据源(这句话不够严谨，后面细讲)。这样一来，同一DAO中的每个方法，便都可以指定数据源了，而背景1需要实现的需求也就迎刃而解。

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
public void invoke(JoinPoint joinPoint) {
        Method method = joinPoint.getCalledMethod();
        // 如果切点处的方法，含有@DataSource注解，则获取其注解值并暂存到DataSourceKeyHolder类中
        if (method != null && method.isAnnotationPresent(DataSource.class)) {
            DataSource dataSourceKey = method.getAnnotation(DataSource.class);
            DataSourceKeyHolder.setDataSourceKey(dataSourceKey);
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

        // 数据源被索取之后，立即重置，防止造成后续数据源的污染
        KEY_HOLDER.set(null);

        return dataSource;
    }
}    
```


未完待续...
