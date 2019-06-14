# 六、SpringMVC实践

> 2019.6.4  端午节前  creaylei

[TOC]

## 1.认识SpringMVC

核心: DispatcherServlet

```java
- Controller   //拦截器

- xxxResolver   //解析器
 - ViewResolver
 - HandlerExceptionResolver
 - MultipartResolver

- HandlerMapping   处理请求映射到  Controller上，即请求映射处理器

//各种常用注解
@Controller  @RestController  @RequestMapping  @PostMapping @GetMapping   @ResponseBody

比较重要的一个注解(之前没见过)  @ResponseStatus(HttpStatus.CREATED)   可以看看 HttpStatus这个类，在方法上面声明。
```

### 1. Spring上下文

下图中的Spring Container就是Spring 的上下文，包含POJOS， 配置等信息

![image-20190611195221863](/Users/zhangleishuidihuzhu.com/Library/Application Support/typora-user-images/image-20190611195221863.png)

**Spring的应用程序上下文**：

**常用的接口**

- BeanFactory    一般不直接用，生产Bean的
  - DefaultListableBeanFactory
- ApplicationContext 扩展了BeanFactory，通常直接使用下面几个
  - ClassPathXmlApplicationContext
  - FileSystemXmlApplicationContext
  - AnnotationConfigApplicationContext
- WebApplicationContext

> java中接口支持多继承的，接口不能实现另一个接口，即不能用implements关键字，而可以extends多个接口。

**上下文的层次关系**

![image-20190611200154967](/Users/zhangleishuidihuzhu.com/Library/Application Support/typora-user-images/image-20190611200154967.png)

Service和Repository等定义在了Root 里

ServletContext 继承 Root Context，当在ServletContext里找不到某个Bean的时候，会到Root Context里面找。

> 引申个问题，如果AOP想拦截一个东西在Root Context里面，而将aop配置在了ServletContext中了，那么拦截就会失效。

这一节中的代码引入一个 @EnableAspectJAutoProxy (开启aop，cs-biz里面也有用)

> java proxy对比cglib：
>
> java proxy由于是java自身支持的，在生成代理类时速度比cglib快，但是cglib是通过插入字节码来实现代理的，在调用速度上cglib要比java proxy快
>
> 在功能上，java proxy仅仅支持接口级别的代理，而cglib没有这个限制，目标类可以是接口可以是普通类，并且可以精确到方法级别（上面的栗子可以看出）。
>
> （摘自某一博客）所以在默认情况下，如果一个目标对象如果实现了接口Spring则会选择JDK动态代理策略动态的创建一个接口实现类（动态代理类）来代理目标对象，可以通俗的理解这个动态代理类是目标对象的另外一个版本，所以这两者之间在强制转换的时候会抛出java.lang.ClassCastException。而所以在默认情况下，如果目标对象没有实现任何接口，Spring会选择CGLIB代理， 其生成的动态代理对象是目标类的子类。
>
> 也可以手动配置一些选项使用spring的CGLIB代理，这样使用CGLIB代理也就不会出现前面提到的ClassCastException问题了，也可以在性能上有所提高，关键是对于代理对象是否继承接口可以统一使用。

这里主要要分清楚，aop要生效具体要给哪里配置，root  还是  servlet

### 2. 配置Web上下文

- xml方式

```xml
<web-app>
	<!--spring监听器-->
	<listener>
			<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>

  <context-param>   <!--全局参数-->
    <param-name>contextConfigLocation</param-name>       //key
    <param-value>/WEB-INF/app-context.xml</param-value>  //value
  </context-param>

  <servlet>   <!--SpringMVC Servlet-->
  	<servlet-name>app</servlet-name>
    <servlet-class>org.springframeword.web.servlet.DispatcherServlet</servlet-class>
    <init-param>    <!--servlet参数-->
    	<param-name>contextConfigLocation</param-name>
      <param-value></param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
  </servlet>

  <servlet-mapping>
  	<servlet-name>app</servlet-name>
    <url-pattern>/app/*</url-pattern>
  </servlet-mapping>

</web-app>


```

- 注解方式

```java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitialize{

	@Override
  protected Class<?>[] getRootConfigClasses() {
    return new Class<?>[] {RootConfig.class};
  }

  @Override
  protected Class<?>[] getServletConfigClasses() {
    return new Class<?>[] {App1Config.class};
  }

  @Overrid
  protected String[] getServletMapping() {
    return new String[] {"/app1/*"};
  }
}
```

[Spring上下文和SpringMVC上下文——源码分析](https://blog.csdn.net/MOVIE14/article/details/78719704)

[参考SpringMVC理解]([http://www.it165.net/pro/html/201502/33644.html](http://www.it165.net/pro/html/201502/33644.html))

---

### 3. SpringMVC中的各种机制

![image-20190612163045717](/Users/zhangleishuidihuzhu.com/Library/Application Support/typora-user-images/image-20190612163045717.png)

[DispatcherServlet初始化过程](https://blog.csdn.net/yangjiachang1203/article/details/51909471)

SpringMVC入口函数就是DispatcherServlet.class   doService调用doDispatch

```java
//1. 一个Controller的请求会先打到
DispatcherServlet.doDispatch(HttpServletRequest request, HttpServletResponse response)

//2. 在doDispatch中先获取对应的Handler
HandlerExecutionChain mappedHandler = getHandler(processedRequest)   //这个方法会遍历所有在Spirng注册的HandlerMapping， handler就是我们常说的Controller控制器
  //这个getHandler方法，里面是取出执行链
HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);

//3. 再拿到对应的HandlerAdapter
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

//4. 调用
ModelAndView mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

//5. 前置和后置的处理
//前置
if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
}
//后置
mappedHandler.applyPostHandle(processedRequest, response, mv);

```

[SpringMVC各部分示意图]([https://github.com/creaylei/JavaGuide/blob/master/%E4%B8%BB%E6%B5%81%E6%A1%86%E6%9E%B6/SpringMVC%20%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E8%AF%A6%E8%A7%A3.md](https://github.com/creaylei/JavaGuide/blob/master/主流框架/SpringMVC 工作原理详解.md))

**一个请求的大致处理流程**

- 绑定一些Attribute
  - WebApplicationContext / LocaleResolver / ThemeResolver
- 处理Multipart
  - 如果是，则将请求转为MultipartHttpServletRequest
- Handler处理
  - 如果找到对应的Handler，执行Controller及前后置处理器逻辑
- 处理返回的Model，呈现视图

---

### 4. 如何定义方法

- **@Controller**
- **@RequestMapping**
  - path  / method    路径和方法
  - params / headers 限定映射范围    带什么头，不带什么头
  - consumes / produces 限定请求与响应格式    接受特定content-type  产生特定content-type
- **一些快捷方式**
  - @RestController
  - @GetMapping/@PostMapping/@PutMapping/@DeleteMapping/@PatchMapping

**定义处理方法**

- @RequestBody请求正文 / @ResponseBody响应正文 / @ResponseStatus
- @PathVariable路径的变量/ **@RequestParam**比较灵活，多看看 /@RequestHeader http头
- HttpEntity / ResponseEntity  接受和响应的

[请求的详细参数官方](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework- reference/web.html#mvc-ann-arguments )

[返回的详细返回官方](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-
reference/web.html#mvc-ann-return-types)

![image-20190612195317027](/Users/zhangleishuidihuzhu.com/Library/Application Support/typora-user-images/image-20190612195317027.png)

```java
//同一个路径，不同的传参，不同的Controller     用到@RequestParam
@GetMapping(path = "/", params = "!name")
    @ResponseBody
    public List<Coffee> getAll() {
        return coffeeService.getAllCoffee();
}
@GetMapping(path = "/", params = "name")
    @ResponseBody
    public Coffee getByName(@RequestParam String name) {
        return coffeeService.getCoffee(name);
}


//动态路径， @PathVariable 厉害了
@GetMapping("/{id}")
    public CoffeeOrder getOrder(@PathVariable("id") Long id) {
        return orderService.get(id);
}
```

### 5. 定义类型转换\校验

- 类型转换：自己实现WebMvcConfigurer
  - Spring Boot在WebMvcAutoConfiguration中实现一个
  - 添加自定义的Converter
  - 添加自定义的Formatter
- 定义校验
  - 通过Validator对绑定结果进行校验
    - Hibernate Validator
  - @Valid注解
  - BindingResult
- Multipart上传(暂时不用)

```java
//第一种，直接加@Valid
public ResponseDto queryReplyContentByType(@RequestParam("typeId") Long typeId)

//第二种，加BindingResult result，然后自己处理异常
   public Coffee addCoffee(@Valid NewCoffeeRequest newCoffee,
                            BindingResult result) {
        if (result.hasErrors()) {
            // 这里先简单处理一下，后续讲到异常处理时会改
            log.warn("Binding Errors: {}", result);
            return null;
        }
        return coffeeService.saveCoffee(newCoffee.getName(), newCoffee.getPrice());
    }
```

### 6. 视图解析器

**ViewResolver**于View接口

- AbstractCachingViewResolver    基类，都是继承这个基类来做的
- UrlBasedViewResolver
- FreeMarkerViewResolver    如果使用了FreeMarker就用这个
- ContentNegotiatingViewResolver   根据返回类型，自动响应，将相应的请求自动分发
- InternalResourceViewResolver



**DispatcherServlet中的视图解析逻辑**

1. 初始化DispatcherServlet的时候会初始化，对应的ViewResolver

- initStrategied()
  - InitViewResolvers()  初始化了对应的ViewResolver
- doDispatch()
  - ProcessDispatchResult()
    - 没有返回视图的话，尝试解析**RequestToViewNameTranslator**
    - resolveViewName() 解析View对象

```java
//servlet中的初始化函数
protected void initStrategies(ApplicationContext context) {   //入参为上下文
		initMultipartResolver(context);
		initLocaleResolver(context);
		initThemeResolver(context);
		initHandlerMappings(context);       //初始化HandlerMapping
		initHandlerAdapters(context);
		initHandlerExceptionResolvers(context);
		initRequestToViewNameTranslator(context);
		initViewResolvers(context);         //初始化ViewResolvers，即这里的视图解析器
		initFlashMapManager(context);
	}

//然后doDispatch方法中
ModelAndView mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException); 呈现/渲染
//processDispatchResult方法里面
render(mv, request, response)

//render里面，调用解析器进行解析
View view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
```

2. 使用@ResponseBody的情况

- 在HandlerAdapter.handle()  中完成了Response的输出
  - RequestMappingHandlerAdapter.invokeHandlerMethod()
    - HandlerMethodReturnValueHandlerComposite.handleReturnValue()
      - RequestResponseBodyMethodProcessor.handleReturnValue()

大概是这四步，建议自己有空再跟一遍断点，看是怎么处理的。



**两种不同重定向：**

- Redirect：客户端行为，url会变
- forward：转发，服务器端行为，url不变

### 7.SpringMVC常用的视图

[官方提供的支持视图说明](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-view) 11种，有配置和使用方式

模板引擎： Thymeleaf、Freemarker、Jackson-based JSON / XML


