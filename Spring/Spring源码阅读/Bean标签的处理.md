# Bean标签的处理

前面部分当标签是 bean 的时候，调用 `processBeanDefinition(ele, delegate)` 

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
        if (bdHolder != null) {
            bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
            try {
                // Register the final decorated instance.
                BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
            }
            catch (BeanDefinitionStoreException ex) {
                getReaderContext().error("Failed to register bean definition with name '" +
                        bdHolder.getBeanName() + "'", ele, ex);
            }
            // Send registration event.
            getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
        }
    }
```

整个过程分为四个步骤

1. 调用 `BeanDefinitionParserDelegate.parseBeanDefinitionElement()` 进行元素解析，解析过程中如果失败，返回 null，错误由 `ProblemReporter` 处理。如果解析成功则返回 BeanDefinitionHolder 实例 bdHolder。BeanDefinitionHolder 为持有 name 和 alias 的 BeanDefinition。
2. 若实例 bdHolder 不为空，则调用 `BeanDefinitionParserDelegate.decorateBeanDefinitionIfRequired()` 进行自定义标签处理
3. 解析完成后，则调用 `BeanDefinitionReaderUtils.registerBeanDefinition()` 对 bdHolder 进行注册
4. 发出响应事件，通知相关的监听器，完成 Bean 标签解析

**其中 `parseBeanDefinitionElement()` 做了如下工作**

这个方法还没有对 Bean 标签进行解析，只是在解析动作之前做了一些功能架构，主要的工作有：

- 解析 id、name 属性，确定 alias 集合，检测 beanName 是否唯一
- 调用方法 `parseBeanDefinitionElement()` 对属性进行解析并封装成 GenericBeanDefinition 实例 beanDefinition
- 根据所获取的信息（beanName、aliases、beanDefinition）构造 BeanDefinitionHolder 实例对象并返回。

*这里有必要说下 beanName 的命名规则* ：如果 id 不为空，则 beanName = id；如果 id 为空，但是 alias 不空，则 beanName 为 alias 的第一个元素，如果两者都为空，则根据默认规则来设置 beanName。 上面三个步骤第二个步骤为核心方法，它主要承担解析 Bean 标签中所有的属性值。如下：

**总流程**

```java
public AbstractBeanDefinition parseBeanDefinitionElement(
            Element ele, String beanName, @Nullable BeanDefinition containingBean) {

        this.parseState.push(new BeanEntry(beanName));

        String className = null;
        // 解析 class 属性
        if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
            className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
        }
        String parent = null;

        // 解析 parent 属性
        if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
            parent = ele.getAttribute(PARENT_ATTRIBUTE);
        }

        try {

            // 创建用于承载属性的 GenericBeanDefinition 实例
            AbstractBeanDefinition bd = createBeanDefinition(className, parent);

            // 解析默认 bean 的各种属性
            parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);

            // 提取 description
            bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

            // 解析元数据
            parseMetaElements(ele, bd);

            // 解析 lookup-method 属性
            parseLookupOverrideSubElements(ele, bd.getMethodOverrides());

            // 解析 replaced-method 属性
            parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

            // 解析构造函数参数
            parseConstructorArgElements(ele, bd);

            // 解析 property 子元素
            parsePropertyElements(ele, bd);

            // 解析 qualifier 子元素
            parseQualifierElements(ele, bd);

            bd.setResource(this.readerContext.getResource());
            bd.setSource(extractSource(ele));

            return bd;
        }
        catch (ClassNotFoundException ex) {
            error("Bean class [" + className + "] not found", ele, ex);
        }
        catch (NoClassDefFoundError err) {
            error("Class that bean class [" + className + "] depends on not found", ele, err);
        }
        catch (Throwable ex) {
            error("Unexpected failure during bean definition parsing", ele, ex);
        }
        finally {
            this.parseState.pop();
        }

        return null;
    }
```

到这里，Bean 标签的所有属性我们都可以看到其解析的过程，也就说到这里我们已经解析一个基本可用的 BeanDefinition。 由于解析过程较为漫长，篇幅较大，为了更好的观看体验，将这篇博文进行拆分

## BeanDefinition

解析Bean标签的过程就是构造一个BeanDefinition对象的过程，`<bean>` 元素标签拥有的配置属性，BeanDefinition 均提供了相应的属性，与之一一对应。所以我们有必要对 BeanDefinition 有一个整体的认识。

### BeanDefinition介绍

BeanDefinition 是一个接口，它描述了一个 Bean 实例，包括属性值、构造方法值和继承自它的类的更多信息。它继承 AttributeAccessor 和 BeanMetadataElement 接口。两个接口定义如下：

- AttributeAccessor ：定义了与其它对象的（元数据）进行连接和访问的约定，即对属性的修改，包括获取、设置、删除。
- BeanMetadataElement：Bean 元对象持有的配置元素可以通过`getSource()` 方法来获取。

![uHRUV1.png](/Users/zhangleishuidihuzhu.com/Pictures/wiznote/uHRUV1.png)

我们常用的三个实现类有：ChildBeanDefinition、GenericBeanDefinition、RootBeanDefinition，三者都继承 AbstractBeanDefinition。如果配置文件中定义了父 `<bean>` 和 子 `<bean>` ，则父 `<bean>` 用 RootBeanDefinition表示，子 `<bean>` 用 ChildBeanDefinition 表示，而没有父 `<bean>` 的就使用RootBeanDefinition 表示。GenericBeanDefinition 为一站式服务类。AbstractBeanDefinition对三个子类共同的类信息进行抽象。

### 解析Bean标签

在最上面的代码中我们可以看到，在 `BeanDefinitionParserDelegate.parseBeanDefinitionElement()` 中完成 Bean 的解析，返回的是一个已经完成对 `<bean>` 标签解析的 BeanDefinition 实例。在该方法内部，首先调用 `createBeanDefinition()` 方法创建一个用于承载属性的 GenericBeanDefinition 实例，如下：

```java
protected AbstractBeanDefinition createBeanDefinition(@Nullable String className, @Nullable String parentName)
			throws ClassNotFoundException {
		return BeanDefinitionReaderUtils.createBeanDefinition(
				parentName, className, this.readerContext.getBeanClassLoader());
	}
```

委托 BeanDefinitionReaderUtils 创建，如下：

```java
public static AbstractBeanDefinition createBeanDefinition(
			@Nullable String parentName, @Nullable String className, @Nullable ClassLoader classLoader) throws ClassNotFoundException {

		GenericBeanDefinition bd = new GenericBeanDefinition();
		bd.setParentName(parentName);
		if (className != null) {
			if (classLoader != null) {
				bd.setBeanClass(ClassUtils.forName(className, classLoader));
			}
			else {
				bd.setBeanClassName(className);
			}
		}
		return bd;
	}
```

该方法主要是设置 parentName 、className、classLoader。

创建完 GenericBeanDefinition 实例后，再调用 `parseBeanDefinitionAttributes()` ，该方法将创建好的 GenericBeanDefinition 实例当做参数，对 Bean 标签的所有属性进行解析，如下：

```java
public AbstractBeanDefinition parseBeanDefinitionAttributes(Element ele, String beanName,
                                                                @Nullable BeanDefinition containingBean, AbstractBeanDefinition bd) {
        // 解析 scope 标签
        if (ele.hasAttribute(SINGLETON_ATTRIBUTE)) {
            error("Old 1.x 'singleton' attribute in use - upgrade to 'scope' declaration", ele);
        }
        else if (ele.hasAttribute(SCOPE_ATTRIBUTE)) {
            bd.setScope(ele.getAttribute(SCOPE_ATTRIBUTE));
        }
        else if (containingBean != null) {
            // Take default from containing bean in case of an inner bean definition.
            bd.setScope(containingBean.getScope());
        }

        // 解析 abstract 标签
        if (ele.hasAttribute(ABSTRACT_ATTRIBUTE)) {
            bd.setAbstract(TRUE_VALUE.equals(ele.getAttribute(ABSTRACT_ATTRIBUTE)));
        }

        // 解析 lazy-init 标签
        String lazyInit = ele.getAttribute(LAZY_INIT_ATTRIBUTE);
        if (DEFAULT_VALUE.equals(lazyInit)) {
            lazyInit = this.defaults.getLazyInit();
        }
        bd.setLazyInit(TRUE_VALUE.equals(lazyInit));

        // 解析 autowire 标签
        String autowire = ele.getAttribute(AUTOWIRE_ATTRIBUTE);
        bd.setAutowireMode(getAutowireMode(autowire));

        // 解析 depends-on 标签
        if (ele.hasAttribute(DEPENDS_ON_ATTRIBUTE)) {
            String dependsOn = ele.getAttribute(DEPENDS_ON_ATTRIBUTE);
            bd.setDependsOn(StringUtils.tokenizeToStringArray(dependsOn, MULTI_VALUE_ATTRIBUTE_DELIMITERS));
        }

        // 解析 autowire-candidate 标签
        String autowireCandidate = ele.getAttribute(AUTOWIRE_CANDIDATE_ATTRIBUTE);
        if ("".equals(autowireCandidate) || DEFAULT_VALUE.equals(autowireCandidate)) {
            String candidatePattern = this.defaults.getAutowireCandidates();
            if (candidatePattern != null) {
                String[] patterns = StringUtils.commaDelimitedListToStringArray(candidatePattern);
                bd.setAutowireCandidate(PatternMatchUtils.simpleMatch(patterns, beanName));
            }
        }
        else {
            bd.setAutowireCandidate(TRUE_VALUE.equals(autowireCandidate));
        }

        // 解析 primay 标签
        if (ele.hasAttribute(PRIMARY_ATTRIBUTE)) {
            bd.setPrimary(TRUE_VALUE.equals(ele.getAttribute(PRIMARY_ATTRIBUTE)));
        }

        // 解析 init-method 标签
        if (ele.hasAttribute(INIT_METHOD_ATTRIBUTE)) {
            String initMethodName = ele.getAttribute(INIT_METHOD_ATTRIBUTE);
            bd.setInitMethodName(initMethodName);
        }
        else if (this.defaults.getInitMethod() != null) {
            bd.setInitMethodName(this.defaults.getInitMethod());
            bd.setEnforceInitMethod(false);
        }

        // 解析 destroy-mothod 标签
        if (ele.hasAttribute(DESTROY_METHOD_ATTRIBUTE)) {
            String destroyMethodName = ele.getAttribute(DESTROY_METHOD_ATTRIBUTE);
            bd.setDestroyMethodName(destroyMethodName);
        }
        else if (this.defaults.getDestroyMethod() != null) {
            bd.setDestroyMethodName(this.defaults.getDestroyMethod());
            bd.setEnforceDestroyMethod(false);
        }

        // 解析 factory-method 标签
        if (ele.hasAttribute(FACTORY_METHOD_ATTRIBUTE)) {
            bd.setFactoryMethodName(ele.getAttribute(FACTORY_METHOD_ATTRIBUTE));
        }
        if (ele.hasAttribute(FACTORY_BEAN_ATTRIBUTE)) {
            bd.setFactoryBeanName(ele.getAttribute(FACTORY_BEAN_ATTRIBUTE));
        }

        return bd;
    }
```

从上面代码我们可以清晰地看到对 Bean 标签属性的解析，这些属性我们在工作中都或多或少用到过。

完成 Bean 标签基本属性解析后，会依次调用 `parseMetaElements()`、`parseLookupOverrideSubElements()`、`parseReplacedMethodSubElements()` 对子元素 meta、lookup-method、replace-method 完成解析。下篇博文将会对这三个子元素进行详细说明。

## [meta、lookup-method、replace-method子元素](http://cmsblogs.com/?p=2736)

[BeanDefinition](http://cmsblogs.com/?p=2734)中已经完成了对 Bean 标签属性的解析工作，这篇博文开始分析子元素的解析。完成 Bean 标签基本属性解析后，会依次调用 、、 对子元素 meta、lookup-method、replace-method 完成解析。三个子元素的作用如下：

- meta：元数据。
- lookup-method：Spring 动态改变 bean 里方法的实现。方法执行返回的对象，使用 Spring 内原有的这类对象替换，通过改变方法返回值来动态改变方法。内部实现为使用 cglib 方法，重新生成子类，重写配置的方法和返回对象，达到动态改变的效果。
- replace-method：Spring 动态改变 bean 里方法的实现。需要改变的方法，使用 Spring 内原有其他类（需要继承接口`org.springframework.beans.factory.support.MethodReplacer`）的逻辑，替换这个方法。通过改变方法执行逻辑来动态改变方法。

> 这里点进去标题链接，详细注意 `loogup-method` 以及`replace-method` 的使用和解析，总的来说还是对数据的预处理，动态覆盖方法，动态改变方法的实现

## [Bean标签之constructor-arg、property 子元素](http://cmsblogs.com/?p=2754)

> 还是最上面的那段代码，解析完上一段对应的三个元素，继续解析 constructor-arg、property、qualifier三个元素

**constructor-arg 举例**

```java
public class StudentService {
    private String name;

    private Integer age;

    private BookService bookService;

    StudentService(String name, Integer age, BookService bookService){
        this.name = name;
        this.age = age;
        this.bookService = bookService;
    }
}

    <bean id="bookService" class="org.springframework.core.service.BookService"/>

    <bean id="studentService" class="org.springframework.core.service.StudentService">
        <constructor-arg index="0" value="chenssy"/>
        <constructor-arg name="age" value="100"/>
        <constructor-arg name="bookService" ref="bookService"/>
    </bean>
```

注意到这里  `<construcor-arg>`  标签

StudentService 定义一个构造函数，配置文件中使用 constructor-arg 元素对其配置，该元素可以实现对 StudentService 自动寻找对应的构造函数，并在初始化的时候将值当做参数进行设置。`parseConstructorArgElements()` 方法完成 constructor-arg 子元素的解析。

这里有  `index、type、name` 三个属性，逐个进行解析

1. 构造 ConstructorArgumentEntry 对象并将其加入到 ParseState 队列中。ConstructorArgumentEntry 表示构造函数的参数。
2. 调用 `parsePropertyValue()` 解析 constructor-arg 子元素，返回结果值
3. 根据解析的结果值构造 `ConstructorArgumentValues.ValueHolder` 实例对象
4. 将 type、name 封装到 `ConstructorArgumentValues.ValueHolder` 中，然后将 ValueHolder 实例对象添加到 indexedArgumentValues 中。

**property子元素**

```xml
 <bean id="studentService" class="org.springframework.core.service.StudentService">
        <property name="name" value="chenssy"/>
        <property name="age" value="18"/>
    </bean>
```

与解析 constructor-arg 子元素步骤差不多。调用 `parsePropertyValue()` 解析子元素属性值，然后根据该值构造 PropertyValue 实例对象并将其添加到 BeanDefinition 中的 MutablePropertyValues 中。

## 解析默认标签下的自定义标签

前面四篇文章都是分析 Bean 默认标签的解析过程，包括基本属性、六个子元素（meta、lookup-method、replaced-method、constructor-arg、property、qualifier），涉及内容较多，拆分成了四篇文章，导致我们已经忘记从哪里出发的了，**勿忘初心**。 `processBeanDefinition()` 负责 Bean 标签的解析（文章最上面的代码），在解析过程中首先调用 `BeanDefinitionParserDelegate.parseBeanDefinitionElement()` 完成默认标签的解析，如果解析成功（返回的 bdHolder != null ），则首先调用 `BeanDefinitionParserDelegate.decorateBeanDefinitionIfRequired()` 完成自定义标签元素解析，前面四篇文章已经分析了默认标签的解析，所以这篇文章分析自定义标签的解析。

```java
 protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
        if (bdHolder != null) {
            bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);

            try {
                BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, this.getReaderContext().getRegistry());
            } catch (BeanDefinitionStoreException var5) {
                this.getReaderContext().error("Failed to register bean definition with name '" + bdHolder.getBeanName() + "'", ele, var5);
            }

            this.getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
        }

    }
```

这里调用自定义标签的解析，注意里面 `bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);` 这个方法就是解析自定义标签

## [解析自定义标签](http://cmsblogs.com/?p=2841)

获取 Document 对象后，会根据该对象和 Resource 资源对象调用 `registerBeanDefinitions()` 方法，开始注册 BeanDefinitions 之旅。在注册 BeanDefinitions 过程中会调用 `parseBeanDefinitions()` 开启 BeanDefinition 的解析过程。在该方法中，它会根据命名空间的不同调用不同的方法进行解析，如果是默认的命名空间，则调用 `parseDefaultElement()` 进行默认标签解析，否则调用 `parseCustomElement()` 方法进行自定义标签解析

### 使用

扩展 Spring 自定义标签配置一般需要以下几个步骤：

1. 创建一个需要扩展的组件
2. 定义一个 XSD 文件，用于描述组件内容
3. 创建一个实现 AbstractSingleBeanDefinitionParser 接口的类，用来解析 XSD 文件中的定义和组件定义
4. 创建一个 Handler，继承 NamespaceHandlerSupport ，用于将组件注册到 Spring 容器
5. 编写 Spring.handlers 和 Spring.schemas 文件

下面就按照上面的步骤来实现一个自定义标签组件。 **创建组件**

## 小节

至此，Bean 的解析过程已经全部完成了，下面做一个简要的总结。 解析 BeanDefinition 的入口在 `DefaultBeanDefinitionDocumentReader.parseBeanDefinitions()``parseDefaultElement()``parseCustomElement()``processBeanDefinition()``processBeanDefinition()`

1. 解析默认标签：`BeanDefinitionParserDelegate.parseBeanDefinitionElement()`
2. 解析默认标签下的自定义标签：`BeanDefinitionParserDelegate.decorateBeanDefinitionIfRequired()`
3. 注册解析的 BeanDefinition：`BeanDefinitionReaderUtils.registerBeanDefinition`

在默认标签解析过程中，核心工作由 `parseBeanDefinitionElement()`

