# Spring官方文档阅读系列

> 抽空看一下官方文档 creaylei 2019-08-23
>
> 参考博文 http://cmsblogs.com/?page_id=3027

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

接口继承示意图

![UTOOLS1566973366858.png](https://i.loli.net/2019/08/28/PJsxAhmWadrBkl6.png)

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

### 1.4依赖

#### Dependency Injection(DI) 依赖注入

> This process is fundamentally the inverse (hence the name, Inversion of Control) of the bean itself controlling the instantiation or location of its dependencies on its own by using direct construction of classes or the Service Locator pattern.

这个过程根本上是控制反转，通过名称，bean自己控制实例化

DI可以是代码更简洁，去耦，最终的效果是，代码简单可测，并且清晰明了。尤其是依赖于接口或者抽象类，这种情况允许在单元测试中使用stud或者mock来实现。

DI有两个不同的变体：

 [Constructor-based dependency injection](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-constructor-injection) 

[Setter-based dependency injection](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-setter-injection) ：spring通过调用bean的无参构造参数，或者无参的静态工厂方法来实例化一个bean，这样来完成基于setter的依赖注入 

> 强制的依赖用 基于构造器的DI (spring建议使用)
>
> 可选的DI，用setter 方法来实现

spring容器完成bean依赖决定的过程:

- 通过描述所有bean的configuration和metadata来创建和实例化applicationContext
- 对于每一个bean，它的依赖被描述为一个个属性，当这个bean被创建的时候，提供这些依赖
- Each property or constructor argument is an actual definition of the value to set, or a reference to another bean in the container

直到bean被创建出来的时候，它的属性才会被容器设置。Beans都是默认单例和预实例化，并且当容器被创建的时候进行实例化这些beans。

#### Lazy-initialized懒加载实例

spring容器创建后，一般的bean都会是 pre-instantiation状态，但是有些bean，完全不用马上加载，比如定时任务，每几个小时，没几天执行一次的，完全可以用lazy标记，告诉springIoC容器，只有当他第一次调用的时候实例化

```xml
<bean id="lazy" class="com.something.ExpensiveToCreateBean" lazy-init="true"/>
<bean name="not.lazy" class="com.something.AnotherBean"/>
```

#### Autowiring Collaborators

**自动注入的优势:**

- 极大的减少构造器注入的参数或者属性
- autowire可以更新你的对象中的配置

自动注入的集中模式

![UTOOLS1566877816361.png](https://i.loli.net/2019/08/27/WhdFymrqUfB6CIa.png)

**缺陷和限制：** 

- 属性或者 constructor-arg的明确配置会覆盖autowire，不能通过autowire来注入String，Classes，这是spring的设计决定的
- 对于有歧义的，会抛出exception

避免被autowired： <bean>标签里面 autowire-candidate 置为false

#### 方法注入method injection

Spring实现通过CGLIB产生字节码，动态的产生一个子类(代理类)来覆盖该方法，这样完成方法注入。

> 这种方式使用的时候，class不能被声明为final

### 1.5Bean Scopes

> spring支持6中scopes应用范围，当使用关于web的applicationContext时候，其中4种是可用的

![UTOOLS1566888194808.png](https://i.loli.net/2019/08/27/Jo9L2sg17GED8aP.png)

- The single scope: 只有一个可共同使用的bean实例，如果被声明单例，springIoc容器只会创建一个bean的实例，然后将其放入一系列单例beans的cache中，后面使用都会返回cache中的实例。使用方式， spring中的单例和设计模式中的单例不一样。

  ```xml
  <bean id="accountService" class="com.something.DefaultAccountService" scope="singleton"/>
  ```

- The protoType(原型设计模式): 每次使用到bean都会创建一个新的实例，即用`getBean()`方法获取

  you should use the prototype scope for all stateful beans and the singleton scope for stateless beans.    有状态的bean使用多实例，无状态的使用单例

  对比其他应用范围，spring不会管理protoType实例的整个生命周期，多实例视为一种`new`的替代

  [为什么默认单例, 单例和原型的对比](https://www.jianshu.com/p/752e5b3b8b94)

- Request, Session, Application, and WebSocket Scopes： 在springMVC中，每一次请求都会由`DispatcherServlet` 来进行处理，不需要其他特殊的操作，`DispatcherServlet`已经暴露了所有相关的状态。`DispatcherServlet`, `RequestContextListener`, and `RequestContextFilter`事实上做了同一件事，即绑定http请求对象到服务该请求的`Thread`上

### 1.6Customizing the nature of bean

> 定制bean的特性，sping提供了大量的接口来定制bean的特性

#### Lifestyle callbacks生命周期回调

为了和spring容器的管理bean生命周期部分交互，可以实现Spring的`InitializingBean` and `DisposableBean`接口。

> 在现代spring应用中，`@PostConstruct` and `@PreDestroy` 被视为获取生命周期回调最好的实现。在内部，spring框架使用 `BeanPostProcessor` 接口处理任何它可以发现的接口回调

对于initialization and destruction callbacks，spring管理对象也要实现`Lifecycle`接口

- Initialization Callbacks： 实现 `InitializingBean` 接口，只有一个方法，这个不建议使用，会增加耦合，建议使用@PostConstruct

  ```
  void afterPropertiesSet() throws Exception;
  ```

- Destruction Callbacks：和上面同样，建议使用 @PreDestroy

  ```
  public class AnotherExampleBean implements DisposableBean {
  
      public void destroy() {
          // do some destruction work (like releasing pooled connections) 做一些销毁后的工作
      }
  }
  ```

当你没有使用 `InitializingBean` and `DisposableBean` callback interfaces，你可以自己写方法，方法名是 `init()`, `initialize()`, `dispose()`, and so on。重写这些方法后，可以不用配置，例如

- 重写init(), <bean>不用配置 init-method="init"

```java
//样例，在bean里面定义init方法，同样的有 initialize(), dispose()这种
public class DefaultBlogService implements BlogService {

    private BlogDao blogDao;

    public void setBlogDao(BlogDao blogDao) {
        this.blogDao = blogDao;
    }

    // this is (unsurprisingly) the initialization callback method
    当服务是null的时候，抛错
    public void init() {
        if (this.blogDao == null) {
            throw new IllegalStateException("The [blogDao] property must be set.");
        }
    }
}
```

这样写完了后，还需要在<beans default-init-method="init"> 配置一下

小节: 有三种行为来控制bean的生命周期

- The [`InitializingBean`](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-factory-lifecycle-initializingbean) and [`DisposableBean`](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-factory-lifecycle-disposablebean) callback interfaces
- Custom `init()` and `destroy()` methods
- The [`@PostConstruct` and `@PreDestroy` annotations](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-postconstruct-and-predestroy-annotations). You can combine these mechanisms to control a given bean.

#### `ApplicationContextAware` and `BeanNameAware`

1 实现了ApplicationContextAware接口的Bean可以获得 applicationContext的引用。 一个用处是回复其他的beans（但是不建议），applicationContext的其他方法包括：提供了文件资源访问、发布应用实践、访问`MessageSource`

```
public interface ApplicationContextAware {
 		//可以设置applicationContext
    void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
}
```

2 实现了BeanNameAware接口的bean，可以获取它关联的对象所定义的name

```
public interface BeanNameAware {

    void setBeanName(String name) throws BeansException;
}
```

其他接口

![UTOOLS1566975242643.png](https://i.loli.net/2019/08/28/2ilXxOYtnW3VojG.png)

> 自己的理解，*Aware相关接口，都是声明相关的接口的接口，或者添加相关属性的

#### 1.8 定制bean

> 开发者不需要实现ApplicationContext的子类。相反的，springIoc容器可以通过插拔一些特别的集成接口的实现来完成扩展。
>
> 使用@BeanPostProcessor来定义bean被spring容器实例化之后的行为。可以实现这个接口来控制实例化的逻辑，依赖逻辑。

使用方式，这样来进行配置

```java
package scripting;

import org.springframework.beans.factory.config.BeanPostProcessor;

public class InstantiationTracingBeanPostProcessor implements BeanPostProcessor {

    // simply return the instantiated bean as-is
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean; // we could potentially return any object reference here...
    }

    public Object postProcessAfterInitialization(Object bean, String beanName) {
        System.out.println("Bean '" + beanName + "' created : " + bean.toString());
        return bean;
    }
}
```

- RequiredAnnotationBeanPostProcessor注解就是BeanPostProcessor的一个实现，用它来确保bean的被这个注解修饰的属性，在被依赖注入的时候有值。

