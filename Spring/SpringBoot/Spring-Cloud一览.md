## Spring-Cloud一览

## 注册中心

spring-cloud支持很多服务发现的软件，Eureka、consul、zookeeper

### Eureka

提供服务的注册和发现功能， 核心注解 @EnableEurekaServer

```yml
服务端配置
server:
  port: 8761

eureka:
  instance:
    hostname: localhost

  client:
    register-with-eureka: false    # 关闭注册 表名自己是一个server
    fetch-registry: false       # 同上
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```



client端配置  注解 @@EnableEurekaClient

```yml
server:
  port: 8762

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/    # 这个就是刚才server的地址

spring:
  application:
    name: first-spring-cloud      # 这个名词非常重要， 在以后服务和服务之间相互调用都是根据这个name
```

### Consul

> 本地可以使用开发模式启动，比较简单，但是没有持久化功能。
>
> 前台启动： consul agent -dev -ui
>
> 后台启动：nohup consul agent -dev -ui > ~/logs/consul.log 2>&1 &
>
> 访问 ： 启动后即可通过本地的8500端口来访问consul了。 [http://localhost:8500](http://localhost:8500/)
>
> 技术选型对比  https://www.consul.io/intro/vs/index.html   Consul vs. Other Software 
>
> consul使用的raft算法

1. 引入依赖，以下两个必须引入

```xml
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-consul-discovery</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
```

2. Application.yml文件

```yml
spring:
  application:
    name: spring-cloud-consul-producer       # 服务名称
  cloud:
    consul:
      host: 127.0.0.1         
      port: 8500
      discovery:
        service-name: service-producer       # 在consul上声明的名称

server:
  port: 8501             # 当前端口
```

3. 启动类：  加上 **@EnableDiscoveryClient**

```java
@EnableDiscoveryClient
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

打开idea多实例选项，分别将server.port  改为 8501、8502启动



然后登陆 http://localhost:8500  发现有两个名字为service-producer的实例



**然后建立一个消费者**

- 引入依赖

```xml
<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-consul-discovery</artifactId>
		</dependency>
```

- 配置yml

```yml
spring:
  application:
    name: spring-cloud-consul-consumer

  cloud:
    consul:
      port: 8500
      host: 127.0.0.1           # consul的地址和端口

      discovery:
        register: false         # 可以注册也可以不用注册

server:
  port: 8503       #当前服务的端口
```

- 启动类

```java
@RestController
public class ServiceController {

    @Autowired
    private LoadBalancerClient loadBalancer;

    @Autowired
    private DiscoveryClient discoveryClient;

		//获取所有的服务
    @RequestMapping("/services")
    public Object services() {
        return discoveryClient.getInstances("service-producer");
    }

  	//对服务service-producer 进行负载均衡
    @RequestMapping("/discover")
    public Object discover() {
        System.out.println(loadBalancer.toString());
        return loadBalancer.choose("service-producer").getUri().toString();
    }
}
```

直接写个controller测试， 其中 loadBalancerClient是 ribbon进行负载均衡的。

**测试**

访问  http://localhost:8503/discover  就会发现，响应是 http://192.168.135.89:8502  和 http://192.168.135.89:8501  轮询出现

## Ribbon: 负载均衡

Spring-Cloud内置的负载均衡

请求打到ribbon发布的端口，然后通过ribbon进行请求，核心代码

```java
@SpringBootApplication
@EnableDiscoveryClient
public class ServiceRibbonApplication {

	public static void main(String[] args) {
		SpringApplication.run(ServiceRibbonApplication.class, args);
	}

	@Bean
	@LoadBalanced
	RestTemplate restTemplate() {
		return new RestTemplate();
	}

}
```

其中 @LoadBanlanced  是负载均衡的注解，通过restTemplate的请求都会进行转发请求

## Hystrix  断路器，熔断器

ribbon集成hystrix

1. 核心依赖

```xml
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
		</dependency>
```

2. 启动类上添加  **@EnableHystrix** 

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableHystrix
public class RibbonApplication {

	public static void main(String[] args) {
		SpringApplication.run(RibbonApplication.class, args);
	}

	@Bean
	@LoadBalanced
	RestTemplate restTemplate() {
		return new RestTemplate();
	}

}
```

3. 要熔断的接口上使用

```java
@Service
public class HelloService {

    @Autowired
    RestTemplate restTemplate;

    @HystrixCommand(fallbackMethod = "hiError")
    public String hiService(String name) {
        return restTemplate.getForObject("http://FIRST-SPRING-CLOUD/hi?name="+name, String.class);
    }

    public String hiError(String name) {
        return "hi," + name + ", sorry, error!";
    }
}
```

这里先通过ribbon进行负载均衡调用  http://FIRST-SPRING-CLOUD/hi?name="+name 方法， 该方法上面使用了@HystrixCommand注解，配合fallbackMethod 熔断调用的方法为 hiError。 如果服务 First-Spring-Cloud挂了，那么直接调用 hiError方法



### 引申下Feign使用Hystrix

Feign自带断路器hystrix，   D版本之后需要在配置文件中开启。

> feign.hystrix.enabled=true

1. 现在声明的Feign接口上配置如下

```java
@FeignClient(
			value = "service-hi",
			fallback = SchedualServiceHiHystric.class)
public interface SchedualServiceHi {
    @RequestMapping(value = "/hi",method = RequestMethod.GET)
    String sayHiFromClientOne(@RequestParam(value = "name") String name);
}
```

value是要调用的服务名， fallback就是配置的熔断类

2. 熔断类的实现

```java
@Component
public class SchedualServiceHiHystric implements SchedualServiceHi {
    @Override
    public String sayHiFromClientOne(String name) {
        return "sorry "+name;
    }
}
```

这里需要使用@Component将其注册到Ioc容器中，  然后实现feign接口中的所有方法，逻辑自己实现就行了。



## Config：实时配置更新

分布式下的各个服务，由于数量多，为了方便统一管理，实时更新，需要配置分布式配置中心组件。。在Spring Cloud中，有分布式配置中心组件spring cloud config ，它支持配置服务放在配置服务的内存中（即本地），也支持放在远程Git仓库中。

在spring cloud config 组件中，分两个角色，

- 一是config server： config server连接的是远程git仓库
- 二是config client。  config client通过 config server读取配置文件

### 创建一个父工程

maven如下

```xml
		<!--项目名称-->
		<groupId>com.forezp</groupId>
    	<artifactId>sc-f-chapter6</artifactId>
    	<version>0.0.1-SNAPSHOT</version>
    <packaging>pom</packaging>
    
		<!--核心配置，分为两个modules, 这个module在另一种架构中也是将项目耦合起来的方式-->
     <modules>
        <module>config-server</module>
        <module>config-client</module>
    </modules>
    
		<!--依赖，项目随便依赖，这里用test举个例子-->
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
```

### 接下来创建 config server和config 

1. 创建子Maven工程，名为**config-server**  pom如下

```xml
	<groupId>com.forezp</groupId>
		<artifactId>config-server</artifactId>
		<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>config-server</name>
	<description>Demo project for Spring Boot</description>

	<!--父项目的地址为上面的gav-->
	<parent>
		<groupId>com.forezp</groupId>
		<artifactId>sc-f-chapter6</artifactId>
		<version>0.0.1-SNAPSHOT</version>
	</parent>

	<!--依赖 config-server-->
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-config-server</artifactId>
		</dependency>
	</dependencies>
```

主程序入口，声明 **@EnableConfigServer** 注解开启配置服务器功能，代码如下：

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConfigServerApplication.class, args);
	}
}
```

在配置文件，application.properties

```properties
spring.application.name=config-server      
server.port=8888

spring.cloud.config.server.git.uri=https://github.com/forezp/SpringcloudConfig/  #git地址
spring.cloud.config.server.git.searchPaths=respo              # 仓库路径
spring.cloud.config.label=master                              # 仓库分支
spring.cloud.config.server.git.username=				#git仓库用户名，公开仓库不用填，私有需要
spring.cloud.config.server.git.password=        #git仓库密码，公开不用， 私有的需要
```

远程仓库https://github.com/forezp/SpringcloudConfig/ 中有个文件config-client-dev.properties文件中有一个属性：

> foo = foo version 3

启动程序：访问http://localhost:8888/foo/dev

```json
{"name":"foo","profiles":["dev"],"label":"master",
"version":"792ffc77c03f4b138d28e89b576900ac5e01a44b","state":null,"propertySources":[]}
```

证明配置服务中心可以从远程程序获取配置信息。

http请求地址和资源文件映射如下:

- /{application}/{profile}[/{label}]
- /{application}-{profile}.yml
- /{label}/{application}-{profile}.yml
- /{application}-{profile}.properties
- /{label}/{application}-{profile}.properties

2. 创建config-client

```xml
	<!--项目名称-->
	<groupId>com.forezp</groupId>
		<artifactId>config-client</artifactId>
		<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>config-client</name>
	<description>Demo project for Spring Boot</description>

	<!--父项目gav-->
	<parent>
		<groupId>com.forezp</groupId>
		<artifactId>sc-f-chapter6</artifactId>
		<version>0.0.1-SNAPSHOT</version>
	</parent>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-config</artifactId>
		</dependency>
	</dependencies>

```

配置文件bootstrap.properties

```properties
spring.application.name=config-client
spring.cloud.config.label=master
spring.cloud.config.profile=dev
spring.cloud.config.uri= http://localhost:8888/
server.port=8881
```

- spring.cloud.config.label 指明远程仓库的分支
- spring.cloud.config.profile
  - dev开发环境配置文件
  - test测试环境
  - pro正式环境
- spring.cloud.config.uri= http://localhost:8888/ 指明配置服务中心的网址。  这里即是我们config-server的地址

程序的入口类，写一个API接口“／hi”，返回从配置中心读取的foo变量的值，代码如下：

```java
@SpringBootApplication
@RestController
public class ConfigClientApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConfigClientApplication.class, args);
	}

	@Value("${foo}")
	String foo;
	@RequestMapping(value = "/hi")
	public String hi(){
		return foo;
	}
}
```

打开网址访问：http://localhost:8881/hi，网页显示：

> foo version 3

这就说明，config-client从config-server获取了foo的属性，而config-server是从git仓库读取的,如图：



## 网关：Zuul 和 Gateway

最新使用的都是Gateway，但是要知道Zuul也是做网关框架的。 下面我们简单看下GateWay的使用

### 简介

Spring Cloud Gateway是Spring Cloud官方推出的**第二代网关框架**，取代Zuul网关。

网关作为流量的，在微服务系统中有着非常作用，网关常见的功能有路由转发、权限校验、限流控制等作用。本文首先用官方的案例带领大家来体验下Spring Cloud的一些简单的功能，在后续文章我会使用详细的案例和源码解析来详细讲解Spring Cloud Gateway.

### 使用

引入依赖

```xml
<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

一个简单的路由

在spring cloud gateway中使用RouteLocator的Bean进行路由转发，将请求进行处理，最后转发到目标的下游服务。在本案例中，会将请求转发到http://httpbin.org:80这个地址上。代码如下：

```java
@SpringBootApplication
@RestController
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
    @Bean
    public RouteLocator myRoutes(RouteLocatorBuilder builder) {
       return builder.routes()
        .route(p -> p
            .path("/get")
            .filters(f -> f.addRequestHeader("Hello", "World"))
            .uri("http://httpbin.org:80"))
        .build();
    } 
}
```

在上面的myRoutes方法中，使用了一个RouteLocatorBuilder的bean去创建路由，除了创建路由RouteLocatorBuilder可以让你添加各种**predicates**和**filters**，predicates断言的意思，顾名思义就是根据具体的请求的规则，由具体的route去处理，filters是各种过滤器，用来对请求做各种判断和修改。

### 集成Hystrix

在spring cloud gateway中可以使用Hystrix。Hystrix是 spring cloud中一个服务熔断降级的组件，在微服务系统有着十分重要的作用。 Hystrix是 spring cloud gateway中是以filter的形式使用的，代码如下：

```java
 @Bean
    public RouteLocator myRoutes(RouteLocatorBuilder builder) {
        String httpUri = "http://httpbin.org:80";
        return builder.routes()
            .route(p -> p
                .path("/get")
                .filters(f -> f.addRequestHeader("Hello", "World"))
                .uri(httpUri))
            .route(p -> p
                .host("*.hystrix.com")
                .filters(f -> f
                    .hystrix(config -> config
                        .setName("mycmd")
                        .setFallbackUri("forward:/fallback")))
                .uri(httpUri))
            .build();
    }
```