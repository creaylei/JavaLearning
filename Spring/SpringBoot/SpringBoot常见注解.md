# SpringBoot注解详解

[TOC]

## 1.注解列表

- @Springbootapplication : 包含了@ComponentScan(让springboot扫描到Configuration类并把他加入程序上下文)、@Configuration和@EnableAutoConfiguration注解。

- @Configuration： **等同于spring的xml配置文件**（自己理解：本来配置spring需要一个xml文件，但是有了这个注解后，改变如下）； 使用java代码可以检查类型安全

  ```java
  //之前
  ApplicationContext context = new classPathXmlApplicationContext(“spring-context.xml”)

  //配置@Configuration之后
  ApplicationContext context = new AnnotationConfigApplicationContext(TestConfiguration.class)    后面是注解配置的那个类名

  //总结(和xml中的配置对比)

   @Configuation等价于<Beans></Beans>
   @Bean等价于<Bean></Bean>
   @ComponentScan等价于<context:component-scan base-package="com.dxz.demo"/>
  ```

  [参考链接](<https://www.cnblogs.com/duanxz/p/7493276.html>)

- @EnableAutoConfiguration自动配置

- @ConponentScan   组件扫描，可自动发现和装配Bean

- @RestController ： 是@Controller + @ResponseBody的合集，表示这是个控制器bean，并且是将函数返回值直接填入http响应体中，是Rest风格的控制器

- @Autowired 自动导入

## 2.注解详解

1. **@SpringBootApplication**： 申明让springBoot进行必要的配置： 用在类上

   ```java
   package com.example.myproject;
   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.autoconfigure.SpringBootApplication;

   @SpringBootApplication // same as @Configuration @EnableAutoConfiguration @ComponentScan
   public class Application {
   public static void main(String[] args) {
   SpringApplication.run(Application.class, args);
   }
   }
   ```

2. **@ResponseBody**：表示该方法的返回结果直接写入HTTP response body中，一般在异步获取数据时使用，用于构建RESTful的api。在使用@RequestMapping后，返回值通常解析为跳转路径，加上@responsebody后返回结果不会被解析为跳转路径，而是直接写入HTTP response body中。比如异步获取json数据，加上@responsebody后，会直接返回json数据。该注解一般会配合@RequestMapping一起使用。示例代码：

   ```java
   @RequestMapping(“/test”)
   @ResponseBody
   public String test(){
   return”ok”;
   }
   ```

3. **@Controller**：用于定义控制器类，在spring 项目中由控制器负责将用户发来的URL请求转发到对应的服务接口（service层），一般这个注解在类中，通常方法需要配合注解@RequestMapping。示例代码：

   ```java
   @Controller
   @RequestMapping(“/demoInfo”)
   publicclass DemoController {
   @Autowired
   private DemoInfoService demoInfoService;

   @RequestMapping("/hello")
   public String hello(Map<String,Object> map){
      System.out.println("DemoController.hello()");
      map.put("hello","from TemplateController.helloHtml");
      //会使用hello.html或者hello.ftl模板进行渲染显示.
      return"/hello";
   	}
   }
   ```

4. **@RequestMapping：**提供路由信息，负责URL到Controller中的具体函数的映射。

5. **@ComponentScan：**表示将该类自动发现扫描组件。个人理解相当于，如果扫描到有@Component、@Controller、@Service等这些注解的类，并注册为Bean，可以自动收集所有的Spring组件，包括@Configuration类。我们经常使用@ComponentScan注解搜索beans，并结合@Autowired注解导入。可以自动收集所有的Spring组件，包括@Configuration类。我们经常使用@ComponentScan注解搜索beans，并结合@Autowired注解导入。如果没有配置的话，Spring Boot会扫描启动类所在包下以及子包下的使用了@Service,@Repository等注解的类。

6. @Repository：使用@Repository注解可以确保DAO或者repositories提供异常转译，这个注解修饰的DAO或者repositories类会被ComponetScan发现并配置，同时也不需要为它们提供XML配置项。

7. @Bean：用@Bean标注方法等价于XML中配置的bean

   ```java
   //之前是需要在spring.xml文件中配置<bean>的，加入这个注解之后，该注解标记在方法上
       // @Bean注解注册bean,同时可以指定初始化和销毁方法
       // @Bean(name="testBean",initMethod="start",destroyMethod="cleanUp")    这个也是管理生命周期
       @Bean
       @Scope("prototype")
       public TestBean testBean() {
           return new TestBean();
       }
   ```



8. @Value：注入**Spring boot application.properties**配置的属性的值。示例代码：

   ```java
   @Value(value = “#{message}”)
   private String message;
   ```

## 3.JPA注解

1. **@Entity**：@Table(name=”“)：表明这是一个实体类。一般用于jpa这两个注解一般一块使用，但是如果表名和实体类名相同的话，@Table可以省略
2. **@MappedSuperClass**： 用在确定是父类的entity上。父类的属性子类可以继承
3. @NoRepositoryBean：一般用作父类的repository，有了这个注解，就不会再去实例化该repository
4. @Column：如果字段名和列名一直，可以省略
5. @Id： 表示该属性是主键
6. **@GeneratedValue**(strategy = GenerationType.SEQUENCE,generator = “repair_seq”)：表示主键生成策略是sequence，(可以为Auto、Identity、native等，Auto表示可以在多个数据库之间切换)，指定sequence的名字为"repair_seq"
7. SequenceGeneretor(name = “repair_seq”, sequenceName = “seq_repair”, allocationSize = 1)：name为sequence的名称，以便使用，sequenceName为数据库的sequence名称，两个名称可以一致。
8. @Transient：表示该属性并非一个到数据库表的字段的映射,ORM框架将忽略该属性。如果一个属性并非数据库表的字段映射,就务必将其标示为@Transient,否则,ORM框架默认其注解为@Basic。
9. @JsonIgnore：作用是json序列化时将[Java ](http://lib.csdn.net/base/java)bean中的一些属性忽略掉,序列化和反序列化都受影响。
10. @JoinColumn(name='loginId')：一对一：本表中指向另外一个表的主键。一对多：另一个表指向本表的主键。
11. OneToOne、@OneToMany、@ManyToOne：对应[hibernate](http://lib.csdn.net/base/javaee)配置文件中的一对一，一对多，多对一

## 4.SpringMVC相关注解

1. @RequestMapping：@RequestMapping(“/path”)表示该控制器处理所有“/path”的UR L请求。RequestMapping是一个用来处理请求地址映射的注解，**可用于类或方法上。**
   用于类上，表示类中的所有响应请求的方法都是以该地址作为父路径，该注解有6个属性
   - params：指定request中必须包含某些参数值，才让该方法处理
   - headers：指定request中必须包含某些指定的header值，才能让该方法处理请求。
   - value:指定请求的实际地址，指定的地址可以是URI Template 模式
   -  method:指定请求的method类型， GET、POST、PUT、DELETE等
   - consumes:指定处理请求的提交内容类型（Content-Type），如application/json,text/html;
   - produces:指定返回的内容类型，仅当request请求头中的(Accept)类型中包含该指定类型才返回

## 5.全局异常处理

- @ControllerAdvice：包含@Component。 可以被扫描到，统一处理异常
- @ExceptionHandler(Exception.class): 用在这个方法上面表示遇到这个异常就执行以下方法

## 6.SpringBoot注解补充

> 这里是不常用的，但是比较实用的

##### @PutMapping = @RequestMapping(method = HttpMethod.PUT)

![UTOOLS1566531946245.png](https://i.loli.net/2019/08/23/jA2QgNkJWqSwKiP.png)

> 这种是可以生成动态的链接

##### @PathVariable 

@PathVariable注解是将方法中的参数绑定到请求URI中的模板变量上。可以通过@RequestMapping注解来指定URI的模板变量，然后使用@PathVariable注解将方法中的参数绑定到模板变量上。特别地，@PathVariable注解允许我们使用value或name属性来给参数取一个别名。下面是使用此注解的一个示例：

![UTOOLS1566532152907.png](https://i.loli.net/2019/08/23/oCwQ5dFrZ6ayD9W.png)



