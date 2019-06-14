# Springboot

> 组件一站式解决方案

[TOC]

## 1. 介绍

### 1. 特点：

- 独立运行：内嵌各种Servlet容器，Tomcat、Jetty等，不需要打成War包部署到容器中，打成一个可独立运行的jar包即可独立运行，所有的依赖包都在一个jar包内。
- 简化配置：spring-boot-starter-web启动器自动依赖其他组件，减少了maven的配置。一系列Starter中就会自动依赖
- 自动配置：可以根据当前类路径下的类、jar包自动配置bean
- 无代码生产和xml配置：springboot配置过程中无代码生成，也无需xml配置文件完成配置工作，这一切都是借助注解完成
- 应用监控：提供一系列端点可以监控服务及应用，做健康监测。

---

### 2.启动的3种方式

1. IDE中启动

   1. jar包方式，IDE中运行main方法启动
   2. war包方式

2. jar包方式  java -jar命令

   > $ java -jar javastack-0.0.1-SNAPSHOT.jar

3. 插件运行

   1. Maven插件

      > mvn spring-boot:run

   2. Gradle Plugin

      > gradle bootRun

### 3. 项目结构介绍

启动类：@SpringBootApplication修饰的启动类，必须包含main方法，然后添加启动方法。

工程布局，application启动类位于根目录下，具体如下

![image-20190611151955997](/Users/zhangleishuidihuzhu.com/Library/Application Support/typora-user-images/image-20190611151955997.png)

**核心注解**：@SpringBootApplication，它主要组合了以下三个注解：

- @SpringBootConfiguration：组合了@Configuration注解，实现配置文件功能
- @EnableAutoConfiguration：打开自动配置，如需关闭某个加参数(exclude = {XXX.class})
- @ComponentScan: spring组件扫面

**配置文件**：

两种：application 和 bootstrap， spring-boot会自动加载classpath目录下的这两个文件，文件格式为 .properties或者.yml格式

> .properties是   key = value
>
> .yml是      key：value   (推荐)

application配置文件：是应用级别的，属于当前项目

bootstrap配置文件: 是系统级别的，用来加载外部文件，如配置中心的配置信息。bootstrap的加载优先于applicaton文件的加载。有一下3个主要场景

- 使用 Spring Cloud Config 配置中心时，这时需要在 bootstrap 配置文件中添加连接到配置中心的配置属性来加载外部配置中心的配置信息；
- 一些固定的不能被覆盖的属性
- 一些加密/解密的场景；

---

### 4. Starter介绍

> Starter可以理解为启动器，包含一系列可以集成到应用中的依赖包。

Starters分类

1. Spring-boot应用类启动器

   | 启动器名称              | 功能描述                                                     |
   | :---------------------- | :----------------------------------------------------------- |
   | spring-boot-starter     | 包含自动配置、日志、YAML的支持。                             |
   | spring-boot-starter-web | 使用Spring MVC构建web 工程，包含restful，默认使用Tomcat容器。 |
   | ...                     | ...                                                          |

2. Spring-boot生产启动器

   | 启动器名称                   | 功能描述                           |
   | :--------------------------- | :--------------------------------- |
   | spring-boot-starter-actuator | 提供生产环境特性，能监控管理应用。 |

3. Spring boot技术类启动器

   | 启动器名称                  | 功能描述                            |
   | :-------------------------- | :---------------------------------- |
   | spring-boot-starter-json    | 提供对JSON的读写支持。              |
   | spring-boot-starter-logging | 默认的日志启动器，默认使用Logback。 |
   | ...                         | ...                                 |

---

### 5. 读取配置的几种方式

**读取aplication文件**

```java
//在application.yml中添加
info.address=china
info.company=Spring
info.degree=high

//第一种，注释到属性上
@Value("${info.address}")
private String address;          //会自动注入

//第二种，注释到类上,直接注入多个
@ConfigurationProperties(prefix = "info")
public class InfoConfig2{
  private String address;
  private String company;
  private String degree;
}
```

**读取指定文件**

```java
//资源目录下建立config/db-config.properties:
db.username=root
db.password=123456

//第一种,@PropertySource+@Value注解，类上指定文件位置，属性上用@Value
  @PropertiesSource(value = {"config/db-config.properties"})
  public class DBconfig1 {
    @Value("${db.username}")
    private String username;
  }

//第二种，@PropertySource+@ConfigurationProperties注解读取方式，类上指定文件位置，同时用@ConfigurationProperties修饰

@PropertiesSource(value = {"config/db-config.properties"})
@ConfigurationProperties(prefix="db")
public class DBconfig2{
  private String username;
  private String password;
}

```

### 6. 延迟加载

Spring框架是支持延迟加载的，就是一个bean不是在项目启动的时候实例化，而是在第一次需要的时候实例化。

提升速度，节省资源。

**传统的Spring项目中这样设置**

```java
<bean id="testBean" calss="cn.javastack.TestBean" lazy-init="true" />
```

Spring-boot中

```java
@Lazy 的使用
```

### 7. 开启定时任务的4中方式

Timer：用的少

**ScheduledExecutorService**：jdk自带的一个类；是基于线程池设计的定时任务类,每个调度任务都会分配到线程池中的一个线程去执行,也就是说,任务是并发执行,互不影响。

**Spring Task**：Spring3.0以后自带的task，可以看成一个轻量级的Quartz，而且使用起来比Quartz简单许多。

**Quartz**：这是一个功能比较强大的的调度器，可以让你的程序在指定时间执行，也可以按照某一个频度执行，配置起来稍显复杂。

主要用第三种

```java
@Scheduled(cron = "")
public vodi scheduled() {
	log.info("hello world!")
}
```

#### Cron表达式

一个cron表达式有至少6个（也可能7个）有空格分隔的时间元素。按顺序依次为：

> 秒（0~59）
>
> 分钟（0~59）
>
> 小时（0~23）
>
> 天（0~31）
>
> 月（0~11）
>
> 星期（1~7 1=SUN 或 SUN，MON，TUE，WED，THU，FRI，SAT）
>
> 年份（1970－2099）

其中每个元素可以是一个值(如6),一个连续区间(9-12),一个间隔时间(8-18/4)(/表示每隔4小时),一个列表(1,3,5),通配符。由于”月份中的日期”和”星期中的日期”这两个元素互斥的,必须要对其中一个设置。推荐：[Spring快速开启计划](http://mp.weixin.qq.com/s?__biz=MzI3ODcxMzQzMw==&mid=2247484266&idx=1&sn=09d18b47002a76b51af95dfff6928ad9&chksm=eb53865cdc240f4a2fd66e2340062c1d04d78cf867abfe35bfac6d7c78bb5c4a11293b8bcd7c&scene=21#wechat_redirect)。

**配置实例：**

> 每隔5秒执行一次：/5 * ?
>
> 每隔1分钟执行一次：0 /1 ?
>
> 0 0 10,14,16 ? 每天上午10点，下午2点，4点
>
> 0 0/30 9-17 ?   朝九晚五工作时间内每半小时
>
> 0 0 12 ? * WED 表示每个星期三中午12点
>
> “0 0 12 ?” 每天中午12点触发
>
> “0 15 10 ? “ 每天上午10:15触发
>
> “0 15 10 ?” 每天上午10:15触发
>
> “0 15 10 ? *” 每天上午10:15触发
>
> “0 15 10 ? 2005” 2005年的每天上午10:15触发
>
> “0 14 * ?” 在每天下午2点到下午2:59期间的每1分钟触发
>
> “0 0/5 14 ?” 在每天下午2点到下午2:55期间的每5分钟触发
>
> “0 0/5 14,18 ?” 在每天下午2点到2:55期间和下午6点到6:55期间的每5分钟触发
>
> “0 0-5 14 ?” 在每天下午2点到下午2:05期间的每1分钟触发
>
> “0 10,44 14 ? 3 WED” 每年三月的星期三的下午2:10和2:44触发
>
> “0 15 10 ? * MON-FRI” 周一至周五的上午10:15触发
>
> “0 15 10 15 * ?” 每月15日上午10:15触发
>
> “0 15 10 L * ?” 每月最后一日的上午10:15触发
>
> “0 15 10 ? * 6L” 每月的最后一个星期五上午10:15触发
>
> “0 15 10 ? * 6L 2002-2005” 2002年至2005年的每月的最后一个星期五上午10:15触发
>
> “0 15 10 ? * 6#3” 每月的第三个星期五上午10:15触发

有些子表达式能包含一些范围或列表

> 例如：子表达式（天（星期））可以为 “MON-FRI”，“MON，WED，FRI”，“MON-WED,SAT”

“*”字符代表所有可能的值
“/”字符用来指定数值的增量

> 例如：在子表达式（分钟）里的“0/15”表示从第0分钟开始，每15分钟
> 在子表达式（分钟）里的“3/20”表示从第3分钟开始，每20分钟（它和“3，23，43”）的含义一样

“？”字符仅被用于天（月）和天（星期）两个子表达式，表示不指定值
当2个子表达式其中之一被指定了值以后，为了避免冲突，需要将另一个子表达式的值设为“？”

“L” 字符仅被用于天（月）和天（星期）两个子表达式，它是单词“last”的缩写
如果在“L”前有具体的内容，它就具有其他的含义了。

> 例如：“6L”表示这个月的倒数第６天
> 注意：在使用“L”参数时，不要指定列表或范围，因为这会导致问题

W 字符代表着平日(Mon-Fri)，并且仅能用于日域中。它用来指定离指定日的最近的一个平日。大部分的商业处理都是基于工作周的，所以 W 字符可能是非常重要的。

> 例如，日域中的 15W 意味着 “离该月15号的最近一个平日。” 假如15号是星期六，那么 trigger 会在14号(星期五)触发，因为星期四比星期一离15号更近。

C：代表“Calendar”的意思。它的意思是计划所关联的日期，如果日期没有被关联，则相当于日历中所有日期。

> 例如5C在日期字段中就相当于日历5日以后的第一天。1C在星期字段中相当于星期日后的第一天。

[参数校验](https://mp.weixin.qq.com/s?__biz=MzI3ODcxMzQzMw==&mid=2247488448&idx=2&sn=766d1d2dfd54f0723a5a1b9efdcbcd7e&chksm=eb5396f6dc241fe00126ebe4039da2bc837bbec8874c330997b66b4160d42f342e2e81294414&scene=21#wechat_redirect)

[集成Swagger](https://mp.weixin.qq.com/s?__biz=MzI3ODcxMzQzMw==&mid=2247488826&idx=2&sn=3e19c0ae6bd58f8583aa0a0d80920459&chksm=eb53900cdc24191a5d8cf34a81e86b49709e7f59ac6d7b5bdddf4a0d80c2fa66edb9802eead7&scene=21#wechat_redirect)

[一份超详细的Spring-boot知识清单](https://mp.weixin.qq.com/s?__biz=MzI3ODcxMzQzMw==&mid=2247487032&idx=1&sn=69ca5adc63684dd38705505ac300ff6b&chksm=eb538b0edc240218124ea1ee6fdc3d044e7d30639fa352333c9076ae56eddb3b9ae87bf30253&scene=21#wechat_redirect) 比较重要

[注册Servlet的三种方法](https://mp.weixin.qq.com/s?__biz=MzI3ODcxMzQzMw==&mid=2247489342&idx=1&sn=20b3163e16f5a6b8abb770d022b22aff&chksm=eb539208dc241b1eb685b3f15ea4d769fb5765344ce2a80b3c901098f5094f4f9fc70e089d8b&scene=21#wechat_redirect)