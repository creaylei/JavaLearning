# Spring源码博客阅读

> [死磕源码](http://cmsblogs.com/?page_id=3027)   chenssy
>
> 类图观看:  
>
> 蓝色实箭头: 继承
>
> 绿色虚线箭头：接口实现
>
> 码云：https://gitee.com/chenssy/blog-home
>
> csdn：https://blog.csdn.net/chenssy

## 1. Spring基础

### IoC简介

Ioc  控制反转，传统的注入是`BeanClass A = new BeanClass()`, 控制反转是由容器来负责对象的生命周期和对象之间的关系。

> IoC的理念，让别人为你服务

![UTOOLS1567308533031.png](https://i.loli.net/2019/09/01/vTtMRzbsml7drWF.png)	三种依赖方式: 

- 构造器注入: 即在bean的构造器中注入

  ```java
  public young{
  	public young(Beutiful beautiful) {
  		this.beautiful = beautiful
  	}
  }
  ```

- setter方法注入: 即通过setter方法注入

  ```
  public young{
  	private Beautifu beautiful;
  	public void setBeautiful(Beautiful beautiful) {
  		this.beautiful = beautiful;
  	}
  }
  ```

- 接口注入，不常用，不介绍

### 各个组件概览

![UTOOLS1567308945916.png](https://i.loli.net/2019/09/01/VxsZwrOlY8g7qzS.png)

这张图包含了，spring容器的IoC体系中大部分核心类和接口

#### 1. Resource体系

#### 2. 有了资源，就应该有资源加载

> 有了资源，就应该有资源加载，Spring 利用 ResourceLoader 来进行统一资源加载，类图如下：

#### 3. BeanFactory体系

> BeanFactory 是一个非常纯粹的 bean 容器，它是 IOC 必备的数据结构，其中 BeanDefinition 是她的基本结构，它内部维护着一个 BeanDefinition map ，并可根据 BeanDefinition 的描述进行 bean 的创建和管理。

#### 4. BeandefinitionReader体系

> BeanDefinitionReader 的作用是读取 Spring 的配置文件的内容，并将其转换成 Ioc 容器内部的数据结构：BeanDefinition。

#### 5. ApplicationContext体系

> 这个就是大名鼎鼎的 Spring 容器，它叫做应用上下文，与我们应用息息相关，她继承 BeanFactory，所以它是 BeanFactory 的扩展升级版，如果BeanFactory 是屌丝的话，那么 ApplicationContext 则是名副其实的高富帅。由于 ApplicationContext 的结构就决定了它与 BeanFactory 的不同，其主要区别有： 1. 继承 MessageSource，提供国际化的标准访问策略。 2. 继承 ApplicationEventPublisher ，提供强大的事件机制。 3. 扩展 ResourceLoader，可以用来加载多个 Resource，可以灵活访问不同的资源。 4. 对 Web 应用的支持。

## 2. 开始死磕Resource

### 统一资源：Resource

spring将资源的定义和资源的加载区分开来，Resource定义了统一的资源，资源的加载由ResourceLoader来统一定义

![image-20190901143047687](/Users/zhangleishuidihuzhu.com/Pictures/wiznote/image-20190901143047687.png)

---

 **AbstractResource** 是`Resource`的默认实现，实现了Resource接口大部分的公共实现，作为Resource接口的重中之重

> 如果我们要实现自定义的Resource，应该采用的方式是继承`AbastractResource`而不是实现Resource

---

### 统一资源定位: ResourceLoader

> Spring将资源的定位和资源的加载区分开了，资源的加载则由 ResourceLoader 来统一定义。统一资源定位器
>
> 查看类图   alt+command+shift+u

![UTOOLS1567319476534.png](https://i.loli.net/2019/09/01/tGTI3uEsFXKAzky.png)

![UTOOLS1567319632802.png](https://i.loli.net/2019/09/01/RC1XfrDhqnpwKyz.png)

ResourceLoader.getResource根据所提供的location来获取资源的实例

getResource支持一下模式资源加载: 

- URL位置资源，如”file:C:/test.dat”
- ClassPath位置资源，如”classpath:test.dat”
- 相对路径资源，如”WEB-INF/test.dat”，此时返回的Resource实例根据实现不同而不同

#### `DefaultResourceLoader`

`DefaultResourceLoader` 是 `ResourceLoader` 的默认实现它接收 ClassLoader 作为构造函数的参数或者使用不带参数的构造函数，在使用不带参数的构造函数时，使用的 ClassLoader 为默认的 ClassLoader（一般为`Thread.currentThread().getContextClassLoader()`），可以通过 `ClassUtils.getDefaultClassLoader()`获取。当然也可以调用 `setClassLoader()`方法进行后续设置。如下：

![image-20190918113925156](/Users/zhangleishuidihuzhu.com/Pictures/wiznote/image-20190918113925156.png)

ResourceLoader 中最核心的方法为 `getResource()`,它根据提供的 location 返回相应的 Resource，而 `DefaultResourceLoader` 对该方法提供了核心实现(它的两个子类都没有提供覆盖该方法，所以可以断定ResourceLoader 的资源加载策略就封装 DefaultResourceLoader中)，如下：

![UTOOLS1568778153098.png](https://img02.sogoucdn.com/app/a/100520146/4cf1a08414ea48098d00a9f3c15864de)

首先通过 ProtocolResolver 来加载资源，成功返回 Resource，否则调用如下逻辑：

- 若 location 以 / 开头，则调用 `getResourceByPath()`构造 ClassPathContextResource 类型资源并返回。
- 若 location 以 classpath: 开头，则构造 ClassPathResource 类型资源并返回，在构造该资源时，通过 `getClassLoader()`获取当前的 ClassLoader。
- 构造 URL ，尝试通过它进行资源定位，若没有抛出 MalformedURLException 异常，则判断是否为 FileURL , 如果是则构造 FileUrlResource 类型资源，否则构造 UrlResource。若在加载过程中抛出 MalformedURLException 异常，则委派 `getResourceByPath()` 实现资源定位加载。

#### `ResourcePatternResolver` : 加载多个资源

ResourceLoader 的 `Resource getResource(String location)` 每次只能根据 location 返回一个 Resource，当需要加载多个资源时，我们除了多次调用 `getResource()` 外别无他法。ResourcePatternResolver 是 ResourceLoader 的扩展，它支持根据指定的资源路径匹配模式每次返回多个 Resource 实例，其定义如下：

![image-20190923141740357](/Users/zhangleishuidihuzhu.com/Pictures/wiznote/image-20190923141740357.png)

ResourcePatternResolver 在 ResourceLoader 的基础上增加了 `getResources(String locationPattern)`，以支持根据路径匹配模式返回多个 Resource 实例，同时也新增了一种新的协议前缀 `classpath*:`，该协议前缀由其子类负责实现。

`PathMatchingResourcePatternResolver`  为 `ResourcePatternResolver`  最常用的子类，它除了支持 ResourceLoader 和 ResourcePatternResolver 新增的 classpath*: 前缀外，还支持 Ant 风格的路径匹配模式（类似于 `**/*.xml`）。

PathMatchingResourcePatternResolver 提供了三个构造方法，如下：

![UTOOLS1569219486753.png](http://yanxuan.nosdn.127.net/8f6b1d93d28dac18e25068fddc309693.png)

PathMatchingResourcePatternResolver 在实例化的时候，可以指定一个 ResourceLoader，如果不指定的话，它会在内部构造一个 DefaultResourceLoader。

**Resource getResource(String location)**

```java
	  @Override
    public Resource getResource(String location) {
        return getResourceLoader().getResource(location);
  	}
```

`getResource()` 方法直接委托给相应的 ResourceLoader 来实现，所以如果我们在实例化的 PathMatchingResourcePatternResolver 的时候，如果不知道 ResourceLoader ，那么在加载资源时，其实就是 DefaultResourceLoader 的过程。其实在下面介绍的 `Resource[] getResources(String locationPattern)` 也相同，只不过返回的资源时多个而已。

**Resource[] getResources(String locationPattern)**

![UTOOLS1569221258698.png](http://yanxuan.nosdn.127.net/a883a80694489e11d72cb579a1c2f35f.png)

处理逻辑如下

![UTOOLS1569221293201.png](http://yanxuan.nosdn.127.net/29657def494a46b80d9ccc46de28f0e5.png)



当 locationPattern 以 classpath*: 开头但是不包含通配符，则调用`findAllClassPathResources()` 方法加载资源。该方法返回 classes 路径下和所有 jar 包中的所有相匹配的资源。

真正执行加载的是在 `doFindAllClassPathResources()`方法，

`doFindAllClassPathResources()` 根据 ClassLoader 加载路径下的所有资源。在加载资源过程中如果，在构造 PathMatchingResourcePatternResolver 实例的时候如果传入了 ClassLoader，则调用其 `getResources()`，否则调用`ClassLoader.getSystemResources(path)`。 `ClassLoader.getResources()`如下:

![UTOOLS1569221876003.png](https://img01.sogoucdn.com/app/a/100520146/583549c2af09cb8dc7a687d88565877f)

看到这里是不是就已经一目了然了？如果当前父类加载器不为 null，则通过父类向上迭代获取资源，否则调用 `getBootstrapResources()`。这里是不是特别熟悉，(*^▽^*)。

> 这里应该就是双亲委派机制，一层一层往上找

**findAllClassPathResources()**

当 locationPattern 以 classpath*: 开头且当中包含了通配符，则调用该方法进行资源加载。如下：

方法有点儿长，但是思路还是很清晰的，主要分两步：

1. 确定目录，获取该目录下得所有资源
2. 在所获得的所有资源中进行迭代匹配获取我们想要的资源。

在这个方法里面我们要关注两个方法，一个是 `determineRootDir()`,一个是 `doFindPathMatchingFileResources()`。

`determineRootDir()`主要是用于确定根路径，如下：

### 小节

至此 Spring 整个资源记载过程已经分析完毕。下面简要总结下：

- Spring 提供了 Resource 和 ResourceLoader 来统一抽象整个资源及其定位。使得资源与资源的定位有了一个更加清晰的界限，并且提供了合适的 Default 类，使得自定义实现更加方便和清晰。
- AbstractResource 为 Resource 的默认实现，它对 Resource 接口做了一个统一的实现，子类继承该类后只需要覆盖相应的方法即可，同时对于自定义的 Resource 我们也是继承该类。
- DefaultResourceLoader 同样也是 ResourceLoader 的默认实现，在自定 ResourceLoader 的时候我们除了可以继承该类外还可以实现 ProtocolResolver 接口来实现自定资源加载协议。
- DefaultResourceLoader 每次只能返回单一的资源，所以 Spring 针对这个提供了另外一个接口 ResourcePatternResolver ，该接口提供了根据指定的 locationPattern 返回多个资源的策略。其子类 PathMatchingResourcePatternResolver 是一个集大成者的 ResourceLoader ，因为它即实现了 `Resource getResource(String location)` 也实现了 `Resource[] getResources(String locationPattern)`。

## 3. Ioc之加载Bean

先来看一段代码，编程式使用Ioc容器

```java
ClassPathResource resource = new ClassPathResource("bean.xml");
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(resource);
```

通过这段代码，可以初步判断IOC容器的使用过程。

- 获取资源
- 获取 BeanFactory
- 根据新建的 BeanFactory 创建一个BeanDefinitionReader对象，该Reader 对象为资源的解析器
- 装载资源 整个过程就分为三个步骤：**资源定位、装载、注册**，如下：

![UTOOLS1569295058277.png](https://img04.sogoucdn.com/app/a/100520146/5bbaef1cc07035e67c98c64192a75ab7)

**资源定位：** 我们一般用外部资源来描述 Bean 对象，所以在初始化 IOC 容器的第一步就是需要定位这个外部资源。在上一篇博客（[【死磕 Spring】—– IOC 之 Spring 统一资源加载策略](http://cmsblogs.com/?p=2656)）已经详细说明了资源加载的过程。

**装载：** 装载就是 BeanDefinition 的载入。BeanDefinitionReader 读取、解析 Resource 资源，也就是将用户定义的 Bean 表示成 IOC 容器的内部数据结构：BeanDefinition。在 IOC 容器内部维护着一个 BeanDefinition Map 的数据结构，在配置文件中每一个 `<bean>` 都对应着一个BeanDefinition对象。

**注册**。向IOC容器注册在第二步解析好的 BeanDefinition，这个过程是通过 BeanDefinitionRegistry 接口来实现的。在 IOC 容器内部其实是将第二个过程解析得到的 BeanDefinition 注入到一个 HashMap 容器中，IOC 容器就是通过这个 HashMap 来维护这些 BeanDefinition 的。在这里需要注意的一点是这个过程并没有完成依赖注入，依赖注册是发生在应用第一次调用 `getBean()` 向容器索要 Bean 时。当然我们可以通过设置预处理，即对某个 Bean 设置 lazyinit 属性，那么这个 Bean 的依赖注入就会在容器初始化的时候完成。 资源定位在前面已经分析了，下面我们直接分析加载，上面提过

`XmlBeanDefinitionReader` 类中的 `reader.loadBeanDefinitions(resource)` 是加载资源的真正实现。

```java
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
        return loadBeanDefinitions(new EncodedResource(resource));
    }
```

从指定的 xml 文件加载 Bean Definition，这里会先对 Resource 资源封装成 EncodedResource。这里为什么需要将 Resource 封装成 EncodedResource呢？主要是为了对 Resource 进行编码，保证内容读取的正确性。封装成 EncodedResource 后，调用

`loadBeanDefinitions()`，这个方法才是真正的逻辑实现。如下

```java
 public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
        Assert.notNull(encodedResource, "EncodedResource must not be null");
        if (logger.isInfoEnabled()) {
            logger.info("Loading XML bean definitions from " + encodedResource.getResource());
        }

        // 获取已经加载过的资源
        Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
        if (currentResources == null) {
            currentResources = new HashSet<>(4);
            this.resourcesCurrentlyBeingLoaded.set(currentResources);
        }

        // 将当前资源加入记录中
        if (!currentResources.add(encodedResource)) {
            throw new BeanDefinitionStoreException(
                    "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
        }
        try {
            // 从 EncodedResource 获取封装的 Resource 并从 Resource 中获取其中的 InputStream
            InputStream inputStream = encodedResource.getResource().getInputStream();
            try {
                InputSource inputSource = new InputSource(inputStream);
                // 设置编码
                if (encodedResource.getEncoding() != null) {
                    inputSource.setEncoding(encodedResource.getEncoding());
                }
                // 核心逻辑部分
                return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
            }
            finally {
                inputStream.close();
            }
        }
        catch (IOException ex) {
            throw new BeanDefinitionStoreException(
                    "IOException parsing XML document from " + encodedResource.getResource(), ex);
        }
        finally {
            // 从缓存中剔除该资源
            currentResources.remove(encodedResource);
            if (currentResources.isEmpty()) {
                this.resourcesCurrentlyBeingLoaded.remove();
            }
        }
    }
```

首先通过

`resourcesCurrentlyBeingLoaded.get()` 来获取已经加载过的资源，然后将 encodedResource 加入其中，如果 resourcesCurrentlyBeingLoaded 中已经存在该资源，则抛出 BeanDefinitionStoreException 异常。完成后从 encodedResource 获取封装的 Resource 资源并从 Resource 中获取相应的 InputStream ，最后将 InputStream 封装为 InputSource 调用 `doLoadBeanDefinitions()`。方法 `doLoadBeanDefinitions()` 为从 xml 文件中加载 Bean Definition 的真正逻辑，如下:

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
            throws BeanDefinitionStoreException {
        try {
            // 获取 Document 实例
            Document doc = doLoadDocument(inputSource, resource);
            // 根据 Document 实例****注册 Bean信息
            return registerBeanDefinitions(doc, resource);
        }catch...
```

核心部分就是 try 块的两行代码。

1. 调用 `doLoadDocument()` 方法，根据 xml 文件获取 Document 实例。
2. 根据获取的 Document 实例注册 Bean 信息。 其实在

`doLoadDocument()`方法内部还获取了 xml 文件的验证模式。如下:

```java
 protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
        return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
                getValidationModeForResource(resource), isNamespaceAware());
    }
```

调用

`getValidationModeForResource()` 获取指定资源（xml）的验证模式。所以 `doLoadBeanDefinitions()`主要就是做了三件事情。

1. 调用 `getValidationModeForResource()` 获取 xml 文件的验证模式

2. 调用 `loadDocument()` 根据 xml 文件获取相应的 Document 实例。

3. 调用 `registerBeanDefinitions()` 注册 Bean 实例。