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

### XML文件的验证模式

> 上面讲到通过调用 `getValidationModeForResource()` 获取 xml 文件的验证模式，这里详细介绍下xml文件的正确性

DTD和XSD的区别

DTD(Document Type Definition)，即文档类型定义，为 XML 文件的验证机制，属于 XML 文件中组成的一部分。DTD 是一种保证 XML 文档格式正确的有效验证方式，它定义了相关 XML 文档的元素、属性、排列方式、元素的内容类型以及元素的层次结构。其实 DTD 就相当于 XML 中的 “词汇”和“语法”，我们可以通过比较 XML 文件和 DTD 文件 来看文档是否符合规范，元素和标签使用是否正确。

要在 Spring 中使用 DTD，需要在 Spring XML 文件头部声明：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC  "-//SPRING//DTD BEAN//EN"  "http://www.springframework.org/dtd/spring-beans.dtd">
```

DTD 在一定的阶段推动了 XML 的发展，但是它本身存在着一些缺陷：

1. 它没有使用 XML 格式，而是自己定义了一套格式，相对解析器的重用性较差；而且 DTD 的构建和访问没有标准的编程接口，因而解析器很难简单的解析 DTD 文档。
2. DTD 对元素的类型限制较少；同时其他的约束力也叫弱。
3. DTD 扩展能力较差。
4. 基于正则表达式的 DTD 文档的描述能力有限。

针对 DTD 的缺陷，W3C 在 2001 年推出 XSD。XSD（XML Schemas Definition）即 XML Schema 语言。XML Schema 本身就是一个 XML文档，使用的是 XML 语法，因此可以很方便的解析 XSD 文档。

`getValidationModeForResource()` 分析

```java
protected int getValidationModeForResource(Resource resource) {
        // 获取指定的验证模式
        int validationModeToUse = getValidationMode();
        // 如果手动指定，则直接返回
        if (validationModeToUse != VALIDATION_AUTO) {
            return validationModeToUse;
        }
        // 通过程序检测
        int detectedMode = detectValidationMode(resource);
        if (detectedMode != VALIDATION_AUTO) {
            return detectedMode;
        }

        // 出现异常，返回 XSD
        return VALIDATION_XSD;
    }
```

如果指定了 XML 文件的的验证模式（调用`XmlBeanDefinitionReader.setValidating(boolean validating)`）则直接返回指定的验证模式，否则调用 `detectValidationMode()` 获取相应的验证模式，如下：

```java
protected int detectValidationMode(Resource resource) {
        if (resource.isOpen()) {
            throw new BeanDefinitionStoreException(
                    "Passed-in Resource [" + resource + "] contains an open stream: " +
                    "cannot determine validation mode automatically. Either pass in a Resource " +
                    "that is able to create fresh streams, or explicitly specify the validationMode " +
                    "on your XmlBeanDefinitionReader instance.");
        }

        InputStream inputStream;
        try {
            inputStream = resource.getInputStream();
        }
        catch (IOException ex) {
            throw new BeanDefinitionStoreException(
                    "Unable to determine validation mode for [" + resource + "]: cannot open InputStream. " +
                    "Did you attempt to load directly from a SAX InputSource without specifying the " +
                    "validationMode on your XmlBeanDefinitionReader instance?", ex);
        }

        try {
          // 核心方法
            return this.validationModeDetector.detectValidationMode(inputStream);
        }
        catch (IOException ex) {
            throw new BeanDefinitionStoreException("Unable to determine validation mode for [" +
                    resource + "]: an error occurred whilst reading from the InputStream.", ex);
        }
    }
```

前面一大堆的代码，核心在于 `this.validationModeDetector.detectValidationMode(inputStream)`，validationModeDetector 定义为 `XmlValidationModeDetector`,所以验证模式的获取委托给 `XmlValidationModeDetector` 的 `detectValidationMode()` 方法。

```java
 public int detectValidationMode(InputStream inputStream) throws IOException {
        BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
        try {
            boolean isDtdValidated = false;
            String content;
            // 一行一行读取 xml 文件的内容
            while ((content = reader.readLine()) != null) {
                content = consumeCommentTokens(content);
                if (this.inComment || !StringUtils.hasText(content)) {
                    continue;
                }
                // 包含 DOCTYPE 为 DTD 模式
                if (hasDoctype(content)) {
                    isDtdValidated = true;
                    break;
                }
                // 读取 < 开始符号，验证模式一定会在 < 符号之前
                if (hasOpeningTag(content)) {
                    // End of meaningful data...
                    break;
                }
            }
        // 为 true 返回 DTD，否则返回 XSD
            return (isDtdValidated ? VALIDATION_DTD : VALIDATION_XSD);
        }
        catch (CharConversionException ex) {
            // 出现异常，为 XSD
            return VALIDATION_AUTO;
        }
        finally {
            reader.close();
        }
    }
```

从代码中看，主要是通过读取 XML 文件的内容，判断内容中是否包含有 DOCTYPE ，如果是 则为 DTD，否则为 XSD，当然只会读取到 第一个 “<” 处，因为 验证模式一定会在第一个 “<” 之前。如果当中出现了 CharConversionException 异常，则为 XSD模式。

好了，XML 文件的验证模式分析完毕，下篇分析 `doLoadBeanDefinitions()` 的第二个步骤：获取 Document 实例。

### Ioc之获取Document对象

`XmlBeanDefinitionReader` 类的

```java
 protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
        return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
                getValidationModeForResource(resource), isNamespaceAware());
    }
```

上面，我们说了`doLoadDocument` 的核心逻辑，这个方法做了两件事，一是调用 `getValidationModeForResource()` 获取 XML 的验证模式，二是调用 `DocumentLoader.loadDocument()` 获取 Document 对象。上篇博客已经分析了获取 XML 验证模式（[【死磕Spring】—– IOC 之 获取验证模型](http://cmsblogs.com/?p=2688)），这篇我们分析获取 Document 对象。 

`DcoumentLoad` 是用来获取document的策略接口，如下 

```java
public interface DocumentLoader {
    Document loadDocument(
            InputSource inputSource, EntityResolver entityResolver,
            ErrorHandler errorHandler, int validationMode, boolean namespaceAware)
            throws Exception;

}

里面的loadDocument方法，由DefaultDocumentLoader实现，如下

@Override
	public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
			ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {

		DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
		if (logger.isTraceEnabled()) {
			logger.trace("Using JAXP provider [" + factory.getClass().getName() + "]");
		}
		DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
		return builder.parse(inputSource);
	}
```

DocumentLoader 中只有一个方法

loadDocument() ，该方法接收五个参数：
* inputSource：加载 Document 的 Resource 源
* entityResolver：解析文件的解析器
* errorHandler：处理加载 Document 对象的过程的错误
* validationMode：验证模式
* namespaceAware：命名空间支持。如果要提供对 XML 名称空间的支持，则为true 该方法由 DocumentLoader 的默认实现类

首先调用

`createDocumentBuilderFactory()` 创建 DocumentBuilderFactory ，再通过该 factory 创建 DocumentBuilder，最后解析 InputSource 返回 Document 对象。

**EntityResolver 通过**

其中有一个参数entityResolver，它是通过`getEntityResolver()` 获取

> `getEntityResolver()` 返回指定的解析器，如果没有指定，则构造一个未指定的默认解析器。

```java
protected EntityResolver getEntityResolver() {
        if (this.entityResolver == null) {
            ResourceLoader resourceLoader = getResourceLoader();
            if (resourceLoader != null) {
                this.entityResolver = new ResourceEntityResolver(resourceLoader);
            }
            else {
                this.entityResolver = new DelegatingEntityResolver(getBeanClassLoader());
            }
        }
        return this.entityResolver;
    }
```

如果 ResourceLoader 不为 null，则根据指定的 ResourceLoader 创建一个 ResourceEntityResolver。如果 ResourceLoader 为null，则创建 一个 DelegatingEntityResolver，该 Resolver 委托给默认的 BeansDtdResolver 和 PluggableSchemaResolver 。

- ResourceEntityResolver：继承自 EntityResolver ，通过 ResourceLoader 来解析实体的引用。
- DelegatingEntityResolver：EntityResolver 的实现，分别代理了 dtd 的 BeansDtdResolver 和 xml schemas 的 PluggableSchemaResolver。
- BeansDtdResolver ： spring bean dtd 解析器。EntityResolver 的实现，用来从 classpath 或者 jar 文件加载 dtd。
- PluggableSchemaResolver：使用一系列 Map 文件将 schema url 解析到本地 classpath 资源

**重要点： `getEntityResolver()` 返回 EntityResolver ，那这个 EntityResolver 到底是什么呢？**

> If a SAX application needs to implement customized handling for external entities, it must implement this interface and register an instance with the SAX driver using the setEntityResolver method.
> 就是说：如果 SAX 应用程序需要实现自定义处理外部实体，则必须实现此接口并使用 `setEntityResolver()` 向 SAX 驱动器注册一个实例。 如下：

```java
public class MyResolver implements EntityResolver {
     public InputSource resolveEntity (String publicId, String systemId){
       if (systemId.equals("http://www.myhost.com/today")){
         MyReader reader = new MyReader();
         return new InputSource(reader);
       } else {
            // use the default behaviour
            return null;
       }
     }
   }
```

我们首先将

`spring-student.xml` 文件中的 XSD 声明的地址改掉，如下：

![UTOOLS1569383303382.png](https://img02.sogoucdn.com/app/a/100520146/88879202e7a63936414d2a4884701968)

如果再次运行程序，会报如下错误

![UTOOLS1569383349937.png](https://img02.sogoucdn.com/app/a/100520146/18424ec11ec2e506fe483e4dc40a95d8)

从上面的错误可以看到，是在进行文档验证时，无法根据声明找到 XSD 验证文件而导致无法进行 XML 文件验证。在([【死磕Spring】—– IOC 之 获取验证模型](http://cmsblogs.com/?p=2688))中讲到，如果要解析一个 XML 文件，SAX 首先会读取该 XML 文档上的声明，然后根据声明去寻找相应的 DTD 定义，以便对文档进行验证。默认的加载规则是通过网络方式下载验证文件，而在实际生产环境中我们会遇到网络中断或者不可用状态，那么就应用就会因为无法下载验证文件而报错。

> EntityResolver作用：应用本身提供一个如何寻找验证文件的方法，即自定义实现。接口声明如下

```java
public interface EntityResolver {
    public abstract InputSource resolveEntity (String publicId,String systemId)
        throws SAXException, IOException;
}
```

接口方法接收两个参数 publicId 和 systemId，并返回 InputSource 对象。两个参数声明如下：

- publicId：被引用的外部实体的公共标识符，如果没有提供，则返回null
- systemId：被引用的外部实体的系统标识符 这两个参数的实际内容和具体的验证模式有关系。如下
- XSD 验证模式
  - publicId：null
  - systemId：http://www.springframework.org/schema/beans/spring-beans.xsd
- DTD 验证模式
  - publicId：-//SPRING//DTD BEAN 2.0//EN
  - systemId：http://www.springframework.org/dtd/spring-beans.dtd 如下：

![UTOOLS1569383524469.png](https://img02.sogoucdn.com/app/a/100520146/395b425f30996acc5351e32627ac1abf)

我们知道在 Spring 中使用 DelegatingEntityResolver 为 EntityResolver 的实现类，`resolveEntity()` 实现如下：

```java
public InputSource resolveEntity(String publicId, @Nullable String systemId) throws SAXException, IOException {
        if (systemId != null) {
            if (systemId.endsWith(DTD_SUFFIX)) {
                return this.dtdResolver.resolveEntity(publicId, systemId);
            }
            else if (systemId.endsWith(XSD_SUFFIX)) {
                return this.schemaResolver.resolveEntity(publicId, systemId);
            }
        }
        return null;
    }
```

不同的验证模式使用不同的解析器解析，如果是 DTD 验证模式则使用 BeansDtdResolver 来进行解析，如果是 XSD 则使用 PluggableSchemaResolver 来进行解析。

```java
 public InputSource resolveEntity(String publicId, @Nullable String systemId) throws IOException {
        if (logger.isTraceEnabled()) {
            logger.trace("Trying to resolve XML entity with public ID [" + publicId +
                    "] and system ID [" + systemId + "]");
        }
        if (systemId != null && systemId.endsWith(DTD_EXTENSION)) {
            int lastPathSeparator = systemId.lastIndexOf('/');
            int dtdNameStart = systemId.indexOf(DTD_NAME, lastPathSeparator);
            if (dtdNameStart != -1) {
                String dtdFile = DTD_NAME + DTD_EXTENSION;
                if (logger.isTraceEnabled()) {
                    logger.trace("Trying to locate [" + dtdFile + "] in Spring jar on classpath");
                }
                try {
                    Resource resource = new ClassPathResource(dtdFile, getClass());
                    InputSource source = new InputSource(resource.getInputStream());
                    source.setPublicId(publicId);
                    source.setSystemId(systemId);
                    if (logger.isDebugEnabled()) {
                        logger.debug("Found beans DTD [" + systemId + "] in classpath: " + dtdFile);
                    }
                    return source;
                }
                catch (IOException ex) {
                    if (logger.isDebugEnabled()) {
                        logger.debug("Could not resolve beans DTD [" + systemId + "]: not found in classpath", ex);
                    }
                }
            }
        }
or wherever.
        return null;
    }
```

`BeansDtdResolver.resolveEntity()` 只是对 systemId 进行了简单的校验（从最后一个 / 开始，内容中是否包含 `spring-beans`），然后构造一个 InputSource 并设置 publicId、systemId，然后返回。

首先调用 getSchemaMappings() 获取一个映射表(systemId 与其在本地的对照关系)，然后根据传入的 systemId 获取该 systemId 在本地的路径 resourceLocation，最后根据 resourceLocation 构造 InputSource 对象。 

### Ioc之注册BeanDefinition

获取 Document 对象后，会根据该对象和 Resource 资源对象调用 `registerBeanDefinitions()` 方法，开始注册 BeanDefinitions 之旅。如下：

```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
        BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
        int countBefore = getRegistry().getBeanDefinitionCount();
        documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
        return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

首先调用 `createBeanDefinitionDocumentReader()` 方法实例化 BeanDefinitionDocumentReader 对象，然后获取统计前 BeanDefinition 的个数，最后调用 `registerBeanDefinitions()` 注册 BeanDefinition。 实例化 BeanDefinitionDocumentReader 对象方法如下：

```java
protected BeanDefinitionDocumentReader createBeanDefinitionDocumentReader() {
        return BeanDefinitionDocumentReader.class.cast(BeanUtils.instantiateClass(this.documentReaderClass));}
```

注册 BeanDefinition 的方法 `registerBeanDefinitions()` 是在接口 BeanDefinitionDocumentReader 中定义，如下

```java
void registerBeanDefinitions(Document doc, XmlReaderContext readerContext)
            throws BeanDefinitionStoreException;
```

**从给定的 Document 对象中解析定义的 BeanDefinition 并将他们注册到注册表中**。方法接收两个参数，待解析的 Document 对象，以及解析器的当前上下文，包括目标注册表和被解析的资源。其中 readerContext 是根据 Resource 来创建的，如下：

DefaultBeanDefinitionDocumentReader 对该方法提供了实现：

```java
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
        this.readerContext = readerContext;
        logger.debug("Loading bean definitions");
        Element root = doc.getDocumentElement();
        doRegisterBeanDefinitions(root);
    }
```

调用 `doRegisterBeanDefinitions()` 开启注册 BeanDefinition 之旅。

程序首先处理 profile属性，profile主要用于我们切换环境，比如切换开发、测试、生产环境，非常方便。然后调用 `parseBeanDefinitions()` 进行解析动作，不过在该方法之前之后分别调用 `preProcessXml()` 和 `postProcessXml()` 方法来进行前、后处理，目前这两个方法都是空实现，交由子类来实现。

`parseBeanDefinitions()` 定义如下：

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
        if (delegate.isDefaultNamespace(root)) {
            NodeList nl = root.getChildNodes();
            for (int i = 0; i < nl.getLength(); i++) {
                Node node = nl.item(i);
                if (node instanceof Element) {
                    Element ele = (Element) node;
                    if (delegate.isDefaultNamespace(ele)) {
                        parseDefaultElement(ele, delegate);
                    }
                    else {
                        delegate.parseCustomElement(ele);
                    }
                }
            }
        }
        else {
            delegate.parseCustomElement(root);
        }
    }
```

最终解析动作落地在两个方法处：`parseDefaultElement(ele, delegate)` `delegate.parseCustomElement(root)`

- 配置文件式声明：`<bean id="studentService" class="org.springframework.core.StudentService"/>`
- 自定义注解方式：`<tx:annotation-driven>`

两种方式的读取和解析都存在较大的差异，所以采用不同的解析方法，如果根节点或者子节点采用默认命名空间的话，则调用 `parseDefaultElement()``delegate.parseCustomElement()``doLoadBeanDefinitions()`

### IoC之核心解析

上一节分析到，有两种解析bean的方式：

- `parseDefaultElement()` 对使用默认命名空间的bean
- `delegate.parseCustomElement` 自定义解析

**整个过程比较清晰：分别是对四种不同的标签进行解析，分别是 import、alias、bean、beans。咱门从第一个标签 import 开始。**

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
       // 对 import 标签的解析
        if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
            importBeanDefinitionResource(ele);
        }
        // 对 alias 标签的解析
        else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
            processAliasRegistration(ele);
        }
        // 对 bean 标签的解析
        else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
            processBeanDefinition(ele, delegate);
        }
        // 对 beans 标签的解析
        else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
            // recurse
            doRegisterBeanDefinitions(ele);
        }
    }
```

#### import标签处理

**获取 source 属性值，得到正确的资源路径，然后调用 loadBeanDefinitions() 方法进行递归的 BeanDefinition 加载**

#### Bean标签处理：重要

> [https://github.com/creaylei/JavaLearning/blob/master/Spring/Spring%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/Bean%E6%A0%87%E7%AD%BE%E7%9A%84%E5%A4%84%E7%90%86.md](https://github.com/creaylei/JavaLearning/blob/master/Spring/Spring源码阅读/Bean标签的处理.md)

本节前面部分当标签是 bean 的时候，调用 `processBeanDefinition(ele, delegate)` 

### [注册BeanDefinition](http://cmsblogs.com/?p=2763)

## 4. [初始化总结](http://cmsblogs.com/?p=2790)

## 5. [开启Bean加载](http://cmsblogs.com/?p=2806)

![UTOOLS1566717296551.png](/Users/zhangleishuidihuzhu.com/Pictures/wiznote/6Ogap25ie4QbWsS.png)

Spring 在实现上述功能中，将整个流程分为两个阶段：容器初始化阶段和加载bean 阶段。

- **容器初始化阶段**：首先通过某种方式加载 Configuration Metadata (主要是依据 Resource、ResourceLoader 两个体系)，然后容器会对加载的 Configuration MetaData 进行解析和分析，并将分析的信息组装成 BeanDefinition，并将其保存注册到相应的 BeanDefinitionRegistry 中。至此，Spring IOC 的初始化工作完成。
- **加载 bean 阶段**：经过容器初始化阶段后，应用程序中定义的 bean 信息已经全部加载到系统中了，当我们显示或者隐式地调用 `getBean()` 时，则会触发加载 bean 阶段。在这阶段，容器会首先检查所请求的对象是否已经初始化完成了，如果没有，则会根据注册的 bean 信息实例化请求的对象，并为其注册依赖，然后将其返回给请求方。至此第二个阶段也已经完成。

第一个阶段，已经通过10多篇文章分析完成。这里开始分析第二个阶段。

详细看：[https://github.com/creaylei/JavaLearning/blob/master/Spring/Spring%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/5%E5%BC%80%E5%90%AFBean%E5%8A%A0%E8%BD%BD.md](https://github.com/creaylei/JavaLearning/blob/master/Spring/Spring源码阅读/5开启Bean加载.md)