# 自己实现一个Starter

> 自己实现SpringBoot里面的Starter

## 原理

一个公用是starter我们只需要引入pom文件，SpringBoot就会进行自动配置。那么SpringBoot是如何知道要实例化哪些类，并进行简单配置呢？

1. 首先，SpringBoot在启动时会去依赖的Starter包中寻找`resources/META-INF/spring.factories `  文件，然后根据文件中配置的Jar包去扫描项目所依赖的Jar包，这类似于 Java 的 **SPI** 机制。

2. 第二步，根据 `spring.factories`配置加载`AutoConfigure`类

3. 最后，根据 `@Conditional`注解的条件，进行自动配置并将Bean注入Spring Context 上下文当中。

   我们也可以使用`@ImportAutoConfiguration({MyServiceAutoConfiguration.class})` 指定自动配置哪些类。

可以认为**starter是一种服务**——使得使用某个功能的开发者不需要关注各种依赖库的处理，不需要具体的配置信息， 由Spring Boot自动通过classpath路径下的类发现需要的Bean，并织入相应的Bean。举个栗子，spring-boot-starter-jdbc这个starter的存在， 使得我们只需要在BookPubApplication下用@Autowired引入DataSource的bean就可以，Spring Boot会自动创建DataSource的实例。

## 核心知识

**1. @Conditional注解**

核心就是条件注解  `@Conditional` ，使用方式如下

```java
//当项目当前Classpath存在 HelloService的时候 后面的配置才生效
@ConditionalOnClass(HelloService.class)
public class HelloServiceAutoConfiguration {}
```

SpringBoot许多源码里面，都会出现条件注解，这就是Starter配置的核心之一。

2. `spring.factories` 文件，这个是springboot启动会扫描的文件，然后寻找所依赖的jar包，写法如下

```java
org.springframework.boot.autoconfigure.EnableAutoConfiguration = 
//等于号后面是定义的starter的路径
com.example.autocinfigure.StarterAutoConfigure
```

## 开发自己的Starter

所谓starter，就是一个普通的Maven项目

我们要实现的starter很简单， 提供一个Service， 包含一个`sayHello()` 方法

主要分为下面几步骤

1. 创建Maven项目
2. 编写service
3. 编写属性类
4. 编写自动配置类
5. 创建spring.factories文件，打包

### 创建Maven工程

新建一个普通Maven项目

- 命名：这里说下artifactId的命名问题，Spring 官方 Starter通常命名为`spring-boot-starter-{name}` 如 `spring-boot-starter-web`。

  Spring官方建议非官方Starter命名应遵循`{name}-spring-boot-starter`的格式。

- 注意其中 `spring-boot-configuration-processor` 的作用是编译时生成`spring-configuration-metadata.json`， 此文件主要给IDE使用，用于提示使用。如在intellij idea中，当配置此jar相关配置属性在`application.yml`， 你可以用ctlr+鼠标左键，IDE会跳转到你配置此属性的类中。

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.xncoding</groupId>
    <artifactId>simple-spring-boot-starter</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>simple-spring-boot-starter</name>
    <description>一个简单的自定义starter</description>
    <url>http://maven.apache.org</url>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <java.version>1.8</java.version>
    </properties>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.4.RELEASE</version>
    </parent>


    <dependencies>
        <!-- @ConfigurationProperties annotation processing (metadata for IDEs)
                 生成spring-configuration-metadata.json类，需要引入此类-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
        </dependency>
    </dependencies>
</project>
```

### 编写Servcie

```java
public class ExampleService {

    private String prefix;
    private String suffix;

    public ExampleService(String prefix, String suffix) {
        this.prefix = prefix;
        this.suffix = suffix;
    }
    public String wrap(String word) {
        return prefix + word + suffix;
    }
}
```

### 编写配置属性类

```java
@ConfigurationProperties("example.service")
public class ExampleServiceProperties {
    private String prefix;
    private String suffix;

    public String getPrefix() {
        return prefix;
    }

    public void setPrefix(String prefix) {
        this.prefix = prefix;
    }

    public String getSuffix() {
        return suffix;
    }

    public void setSuffix(String suffix) {
        this.suffix = suffix;
    }
}
```

其中`@ConfigurationPropeties` 注解是读取 application.yml中 以 example.service 开头的属性

### 编写自动配置类

```java
@Configuration
@ConditionalOnClass(ExampleService.class)
@EnableConfigurationProperties(ExampleServiceProperties.class)
public class ExampleAutoConfigure {

    private final ExampleServiceProperties properties;

    @Autowired
    public ExampleAutoConfigure(ExampleServiceProperties properties) {
        this.properties = properties;
    }

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(prefix = "example.service", value = "enabled",havingValue = "true")
    ExampleService exampleService (){
        return  new ExampleService(properties.getPrefix(),properties.getSuffix());
    }

}
```

解释下用到的几个和Starter相关的注解：

```java
1. @ConditionalOnClass，当classpath下发现该类的情况下进行自动配置。
2. @ConditionalOnMissingBean，当Spring Context中不存在该Bean时。
3. @ConditionalOnProperty(prefix = "example.service",value = "enabled",havingValue = "true")，当配置文件中example.service.enabled=true时。
```

### 最后一步：添加spring.factories文件

最后一步，在`resources/META-INF/`下创建`spring.factories`文件，内容供参考下面：

```java
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.xncoding.starter.config.ExampleAutoConfigure
```

如果有多个自动配置类，用逗号分隔换行即可。

OK，完事，运行 `mvn:install` 打包安装，一个Spring Boot Starter便开发完成了。



## 测试

打包好了当然要测试一下看看了。另外创建一个SpringBoot工程，在maven中引入这个starter依赖， 然后在单元测试中引入这个Service看看效果。

```xml
<dependency>
    <groupId>com.xncoding</groupId>
    <artifactId>simple-spring-boot-starter</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

修改`application.yml`配置文件，添加如下内容：

```xml
example.service:
  enabled: true
  prefix: ppp
  suffix: sss
```

测试类

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class ApplicationTests {
    @Autowired
    private ExampleService exampleService;

    @Test
    public void testStarter() {
        System.out.println(exampleService.wrap("hello"));
    }
}
```

运行结果

```java
ppphellosss
```

