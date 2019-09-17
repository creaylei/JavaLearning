# Guava

> Uset Guide https://github.com/google/guava/wiki  
>
> 官方项目地址: https://github.com/google/guava
>
> 中文非官方项目 : http://ifeve.com/google-guava/#more-8776

学习常用的学习，有些暂时用不到跳过

[TOC]

## 1.基本工具

### 1.2 前置条件判断

`Preconditions` 类

Guava在[Preconditions](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/base/Preconditions.html)类中提供了若干前置条件判断的实用方法，我们强烈建议[在Eclipse中静态导入这些方法](http://ifeve.com/eclipse-static-import/)。每个方法都有三个变种：

- 没有额外参数：抛出的异常中没有错误消息；
- 有一个Object对象作为额外参数：抛出的异常使用Object.toString() 作为错误消息；
- 有一个String对象作为额外参数，并且有一组任意数量的附加Object对象：这个变种处理异常消息的方式有点类似printf，但考虑GWT的兼容性和效率，只支持%s指示符。

### 1.3 常见Objects方法

#### Objects.equals

guava中使用的是  `Objects.equal` , 但是注意  jdk7开始引入  `Obejects.equals`

#### Objects.hashCode

Guava:   `Objects.hashCode(Object...)`

JDK7:    `Objects.hash(Object...)`

#### Objects.toString

Objects.toStringHelper : 

```java
1 // Returns "ClassName{x=1}"
2 Objects.toStringHelper(this).add("x", 1).toString();
3 // Returns "MyObject{x=1}"
4 Objects.toStringHelper("MyObject").add("x", 1).toString();
```

#### ComparisonChain

一定要会用，下面是实现compare接口中复写compareTo方法

![UTOOLS1568711062409.png](https://img02.sogoucdn.com/app/a/100520146/c9f17acf626ff64a48b6847d79a45d57)

这种[Fluent接口](http://en.wikipedia.org/wiki/Fluent_interface)风格的可读性更高，发生错误编码的几率更小，并且能避免做不必要的工作。

### 1.4 排序

## 2.集合Collections

> 这一部分是最成熟和为人所知的部分

### 2.1 不可变集合

> 为甚么要用不可变集合
>
> - 当对象被不可信的库调用时，不可变形式是安全的；
> - 不可变对象被多个线程调用时，不存在竞态条件问题
> - 不可变集合不需要考虑变化，因此可以节省时间和空间。所有不可变的集合都比它们的可变形式有更好的内存利用率（分析和测试细节）；
> - 不可变对象因为有固定不变，可以作为常量来安全使用。

具体再了解

### 2.3 强大的集合工具类

jdk和guava集合类关系对比

![UTOOLS1568712181052.png](https://img04.sogoucdn.com/app/a/100520146/2627c007afdef2ee4fe7c846279b5817)

#### 静态工厂方法

Lists.newArrayList();

Lists.newArrayList("a","b","c");

Lists.newArrayListWithCapacity(100);

#### Iterables

guava偏向于对iterable处理

> Guava提供的工具方法更偏向于接受Iterable而不是Collection类型。在Google，对于不存放在主存的集合——比如从数据库或其他数据中心收集的结果集，因为实际上还没有攫取全部数据，这类结果集都不能支持类似size()的操作 ——通常都不会用Collection类型来表示。