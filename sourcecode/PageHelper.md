# PageHelper

`PageHelper`是常用的分页插件，之前只停留在用的阶段，直到.......

发生了一次生产事故

所以准备把他的源码研究一下，看下原因

## 介绍(What)

![image-20200530110229737](https://tva1.sinaimg.cn/large/007S8ZIlly1gfaao6zv5sj317m0f80vq.jpg)

从这里看到，`PageHelper`的使用是建立在`Mybatis`基础之上的，具体实现后面再说。 同时支持常见的各种数据库如 Mysql、Oracle、H2等，基本都能支持。





## 使用(How)

### 重要提示

#### `PageHelper.startPage`方法重要提示:

只有紧跟在`PageHelper.startPage`方法后的**第一个**Mybatis的**查询（Select）**方法会被分页。

#### 请不要配置多个分页插件

请不要在系统中配置多个分页插件(使用Spring时,`mybatis-config.xml`和`Spring<bean>`配置方式，请选择其中一种，不要同时配置多个分页插件)！

#### 分页插件不支持带有`for update`语句的分页

对于带有`for update`的sql，会抛出运行时异常，对于这样的sql建议手动分页，毕竟这样的sql需要重视。

#### 分页插件不支持嵌套结果映射

由于嵌套结果方式会导致结果集折叠，因此分页查询的结果折叠后总数会减少，所以无法保证分页结果数量正确。

> 初步怀疑我的问题是这一步引起，因为我是联表查询

---

### 1. 引入Maven依赖

```xml
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper</artifactId>
    <version>最新版本</version>
</dependency>
```

### 2. 配置拦截器

> 分页插件的使用是建立在 Mybatis拦截器上的，执行sql前会进行参数拼接

在 MyBatis 配置 xml 中配置拦截器插件

```
<plugins>
    <!-- com.github.pagehelper为PageHelper类所在包名 -->
    <plugin interceptor="com.github.pagehelper.PageInterceptor">
        <!-- 使用下面的方式配置参数，后面会有所有的参数介绍 -->
        <property name="param1" value="value1"/>
	</plugin>
</plugins>
```

### 3. 使用

最常见的用法

```
//获取第1页，10条内容，默认查询总数count
PageHelper.startPage(1, 10);
//紧跟着的第一个select方法会被分页
List<User> list = userMapper.selectIf(1);
PageInfo page = new PageInfo(list);
```

---

## 原理(Why)

今天就来对源码一探究竟

### PageHelper

从这个类开始

```
PageHelper.startPage(1,10);
```

我们点进去看看

```java
		/**
     * 开始分页
     *
     * @param pageNum  页码
     * @param pageSize 每页显示数量
     */
    public static <E> Page<E> startPage(int pageNum, int pageSize) {
        return startPage(pageNum, pageSize, true);
    }
    
    //多了一个是否进行count查询的参数，由上面看，默认开启
    public static <E> Page<E> startPage(int pageNum, int pageSize, boolean count) {
        return startPage(pageNum, pageSize, count, null);
    }

		//多了个 reasonable : 分页合理化
		public static <E> Page<E> startPage(int pageNum, int pageSize, boolean count, Boolean reasonable) {
        return startPage(pageNum, pageSize, count, reasonable, null);
    }

		//终于到方法体了，多了个 pageSizeZero
		//这样介绍的  你可以配置 pageSizeZero 为 true， 配置后，当 pageSize=0 或者 RowBounds.limit = 0 就会查询出全部的结果。
		public static <E> Page<E> startPage(int pageNum, int pageSize, boolean count, Boolean reasonable, Boolean pageSizeZero) {
        Page<E> page = new Page<E>(pageNum, pageSize, count);   //新建page类
        page.setReasonable(reasonable);
        page.setPageSizeZero(pageSizeZero);                    //下面这俩参数默认没有
        //当已经执行过orderBy的时候
        Page<E> oldPage = SqlUtil.getLocalPage();
        if (oldPage != null && oldPage.isOrderByOnly()) {
            page.setOrderBy(oldPage.getOrderBy());
        }
        SqlUtil.setLocalPage(page);
        return page;
    }
```

流程就是新建一个Page类，主要给SqlUtil设置  `LocalPage()`, 看下LocalPage()实现

```
public class SqlUtil implements Constant {
    private static final ThreadLocal<Page> LOCAL_PAGE = new ThreadLocal<Page>();
}
```

是一个线程私有的 ThreadLocal，这个方法到此结束。

那么，设置完这个，还不知道怎么执行分页的。我们再回到PageHelper

### 怎么生效的

回到PageHelper后，我们看到 pageHelper 实现了 mybatis的拦截器

```java
public class PageHelper implements Interceptor {

}

//该接口如下
package org.apache.ibatis.plugin;

import java.util.Properties;
public interface Interceptor {
    Object intercept(Invocation var1) throws Throwable;

    Object plugin(Object var1);

    void setProperties(Properties var1);
}
```

那么可以断言，在Mybatis执行的时候，PageHelper通过拦截器做了些工作，我们来看 pageHelper中的`intercept`方法

```java
/**
     * Mybatis拦截器方法
     *
     * @param invocation 拦截器入参
     * @return 返回执行结果
     * @throws Throwable 抛出异常
     */
    public Object intercept(Invocation invocation) throws Throwable {
        if (autoRuntimeDialect) {      //自动检测 数据库类型
            SqlUtil sqlUtil = getSqlUtil(invocation);
            return sqlUtil.processPage(invocation);
        } else {
            if (autoDialect) {
                initSqlUtil(invocation);
            }
            return sqlUtil.processPage(invocation);
        }
    }
```

注释非常清晰，核心是  `sqlUtil.processPage` 方法，继续看

```java
/**
     * Mybatis拦截器方法，这一步嵌套为了在出现异常时也可以清空Threadlocal
     *
     * @param invocation 拦截器入参
     * @return 返回执行结果
     * @throws Throwable 抛出异常
     */
    public Object processPage(Invocation invocation) throws Throwable {
        try {
            Object result = _processPage(invocation);
            return result;
        } finally {
            clearLocalPage();
        }
    }
```

继续看 `_processPage()`

![image-20200530120547485](https://tva1.sinaimg.cn/large/007S8ZIlly1gfaci20wu6j31c10u0gsu.jpg)

图中1是指：

> `supportMethodsArguments`：支持通过 Mapper 接口参数来传递分页参数，默认值`false`，分页插件会从查询方法的参数值中，自动根据上面 `params` 配置的字段中取值，查找到合适的值时就会自动分页。

2是真正处理的方法，我们放他出来, 比较长，但是耐心看看。

```java
/**
     * Mybatis拦截器方法
     *
     * @param invocation 拦截器入参
     * @return 返回执行结果
     * @throws Throwable 抛出异常
     */
    private Page doProcessPage(Invocation invocation, Page page, Object[] args) throws Throwable {
        //保存RowBounds状态
        RowBounds rowBounds = (RowBounds) args[2];
        //获取原始的ms，   这里即  包名 + 方法名定位的sql语句
        MappedStatement ms = (MappedStatement) args[0];
        //判断并处理为PageSqlSource
        if (!isPageSqlSource(ms)) {
            processMappedStatement(ms);
        }
        //设置当前的parser，后面每次使用前都会set，ThreadLocal的值不会产生不良影响
        ((PageSqlSource)ms.getSqlSource()).setParser(parser);
        try {
            //忽略RowBounds-否则会进行Mybatis自带的内存分页
            args[2] = RowBounds.DEFAULT;
            //如果只进行排序 或 pageSizeZero的判断
            if (isQueryOnly(page)) {
                return doQueryOnly(page, invocation);
            }

            //简单的通过total的值来判断是否进行count查询
            if (page.isCount()) {
                page.setCountSignal(Boolean.TRUE);
                //替换MS
                args[0] = msCountMap.get(ms.getId());
                //查询总数
                Object result = invocation.proceed();
                //还原ms
                args[0] = ms;
                //设置总数
                page.setTotal((Integer) ((List) result).get(0));
                if (page.getTotal() == 0) {
                    return page;
                }
            } else {
                page.setTotal(-1l);
            }
            //pageSize>0的时候执行分页查询，pageSize<=0的时候不执行相当于可能只返回了一个count
            if (page.getPageSize() > 0 &&
                    ((rowBounds == RowBounds.DEFAULT && page.getPageNum() > 0)
                            || rowBounds != RowBounds.DEFAULT)) {
                //将参数中的MappedStatement替换为新的qs
                page.setCountSignal(null);
                BoundSql boundSql = ms.getBoundSql(args[1]);
  //注意这里，是获取 pageStartRow 和  pageSize
                args[1] = parser.setPageParameter(ms, args[1], boundSql, page); 
              
  //之前已经执行过count()查询
                page.setCountSignal(Boolean.FALSE);
                //执行分页查询
                Object result = invocation.proceed();
                //得到处理结果
                page.addAll((List) result);
            }
        } finally {
            ((PageSqlSource)ms.getSqlSource()).removeParser();
        }

        //返回结果
        return page;
    }
```

1. 先查判断是否查总数，具体是通过page.isCount(), 这个参数我们在前面默认设置为true，
2. 最后将执行的结果放进Page，page也是 ArrayList

### 题外

这里注意下

![image-20200530122930469](https://tva1.sinaimg.cn/large/007S8ZIlly1gfad6q2crgj31ke0o67dx.jpg)

这个方法使用了桥接模式，抽象类实现了基本的方法

![image-20200530123041111](https://tva1.sinaimg.cn/large/007S8ZIlly1gfad7y7ffcj30k40cstap.jpg)

每个数据库继承了此类，实现了自己的方法

![image-20200530123219847](https://tva1.sinaimg.cn/large/007S8ZIlly1gfad9nzp6sj31fc0f8jsb.jpg)

我们来看下 `MysqlParser`

![image-20200530123335016](https://tva1.sinaimg.cn/large/007S8ZIlly1gfadb57j6dj31nw0o043r.jpg)

主要是设置参数，和分页查询sql。

## 小节

PageHelper的原理主要是 mybatis拦截器

查总数用 select count(0) from ( 自己的sql)

分页用    select  ----   limit  ？，？ 实现分页

后面放到 PageInfo里面去