# Spring官方文档阅读系列

> 抽空看一下官方文档 creaylei 2019-08-23

[TOC]

## 1. The IoC Container

ioc容器的基础包: org.springframework.beans 和 org.springframework.context

<u>BeanFactory</u>： 可以管理任何 Object的接口

- `ApplicationContext` 是 BeanFactory的一个子接口，他提供了
  1. springAOP的集成
  2. 消息资源的处理
  3. 事件发布
  4. 应用层准确的上下文，如 `WebApplicationContext` for use in web applications

ApplicationContext是BeanFactory的一个超集，[beans-beanfactory](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-beanfactory) 详情

spring提供了几个ApplicationContext的实现

```
org.springframework.context.support.ClassPathXmlApplicationContext

org.springframework.context.support.FileSystemXmlApplicationContext
```

### 1.2容器综述

#### 元数据配置

Spring工作原理

![UTOOLS1566717296551.png](https://i.loli.net/2019/08/25/6Ogap25ie4QbWsS.png)

在springContainer创建后，POJO类通过和metadata连接，就形成一个完整的应用和可执行系统

- 配置元数据: spring IoC容器使用了一系列元数据，通过元数据，你可以告诉spring容器怎样去实例化，配置和装配应用，分为

  -  xml： 使用 

    ```
    <beans>
    	<bean id="" class="">
    	</bean>
    ```

    

  - annotation-based configuration，

  -  java-based configuration： @Configuration， @Bean @Import

#### 容器实例化

定位路径提供了一个`ApplicationContext`构造器，允许容器加载外部配置元数据

```java
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");
```

#### 容器使用

ApplicationContext是一个支持多个不同实例，以及他们的依赖的工厂接口。

> The `ApplicationContext` is the interface for an advanced factory capable of maintaining a registry of different beans and their dependencies.         By using the method `T getBean(String name, Class<T> requiredType)`, you can retrieve instances of your beans.

举例

```java
// create and configure beans
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");

// retrieve configured instance
PetStoreService service = context.getBean("petStore", PetStoreService.class);

// use configured instance
List<String> userList = service.getUsernameList();
```

### 1.3Bean综述

bean的定义

![UTOOLS1566719956606.png](https://i.loli.net/2019/08/25/zxlCFZRMuQ7egao.png)

#### Naming Beans（bean的命名）

> 同一个容器下尽量唯一，如果需要多个名字，考虑使用别名

- 别名可以在xml中使用<alias name= "fileName" alias="toName">
- Java代码中可以使用@Bean  (name = "")

### 1.4依赖(还没看)

