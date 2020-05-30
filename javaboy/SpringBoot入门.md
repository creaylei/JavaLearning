# JAVABOY（一）

> Javaboy  学习，微人事



## 查漏补缺

```xml
<groupId>com.example</groupId>
	<artifactId>demo</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>war</packaging>
```

这里，packaging声明将要打成war包。



Spring父容器负责所有其他非@Controller注解的Bean的注册，而SpringMVC只负责@Controller注解的Bean的注册，使得他们各负其责、明确边界。



@SpringBootApplication, 默认扫描当前启动类所在的包，底下的所有文件。  组合@SpringConfiguration，@EnableAutoConfiguration 自动配置类。   然后就是@ComponentScan扫描范围



## 2 SpringBoot基础配置

### 2.1 Banner配置

设置Banner：resources——> 新建 banner.txt ——> 里面的内容就是banner

关闭Banner

1. application.yml文件中这样写

```
spring:
  main:
    banner-mode: "off"
```

2. 在主类中通过代码更改

### 2.2 容器相关配置

**查看内嵌tomcat**：

1. 新建一个springboot项目，maven中web依赖点开后，会有一个tomcat的starter，就是这里

2. 修改服务器端口号：在application.yml中加入

   ```
   server:
   	port: 8081
   ```

3. 加入上下文环境，加入后访问 localhost:8080/javaboy/+++++

   ```yml
   server:
     servlet:
       context-path: /javaboy
   ```

> 其他容器：jetty、undertow  都比较新，可以尝试

### 2.3 Springboot属性注入

application.yml文件中可以定义属性的值，如

```yml
book.id = 99
book.name=三国演义
book.author=罗贯中
```

然后再Bean中这样用，$ 符号和中括号，在加上yml中的变量名：

```
@Component
public class Book{

	@Value("${book.name}")
	private String name;
}
```

> 因为实际应用中属性注入的比较多，那么都放在application.yml中可能不太友好
>
> 这时候，新建在resources下新建，book.properties
>
> 然后再在bean上加上注解，   @PropertySource("classpath:book.properties")

### 2.4 Springboot提供的注解

上面2.3的注解都是springframework提供的，也就是原生的。 而springboot也提供了一个非常有用的注解

@ConfigurationProperties， 用法

```java
@ConfigurationProperties(prefix='book')
public class Book{

	//@Value("${book.name}")
	private String name;
}
```

加上了prefix之后， 后面的都会直接对应到属性名上，即属性上不用加注解了

### 2.5 yml/yaml文件格式

```yml
redis:
	port: 6739
	hosts: 
		- 192.168.1.1
		- 192.168.66.129      
```

加 “-” 表示新的，也可以引入对象

### 2.6 profile区分环境

新建application-dev.yml    application-test.yml、  application-prod.yml

在application.yml中指定

```yml
spring:
	profiles: 
		active: dev
```

这样就指定了环境