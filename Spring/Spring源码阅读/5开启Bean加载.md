# 5开启Bean加载

从这篇开始分析第二个阶段：加载 bean 阶段。当我们显示或者隐式地调用 `getBean()` 时，则会触发加载 bean 阶段。如下：

```java
public Object getBean(String name) throws BeansException {
        return doGetBean(name, null, null, false);
    }
```

内部调用 `doGetBean()`

- name：要获取 bean 的名字
- requiredType：要获取 bean 的类型
- args：创建 bean 时传递的参数。这个参数仅限于创建 bean 时使用
- typeCheckOnly：是否为类型检查

这里doGetBean代码较长

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
                              @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

        // 获取 beanName，这里是一个转换动作，将 name 转换为 beanName,
  			// 这两个name之间的区别是  是不是以 & 开头
        final String beanName = transformedBeanName(name);
        Object bean;

        // 从缓存中或者实例工厂中获取 bean
        // *** 这里会涉及到解决循环依赖 bean 的问题
        Object sharedInstance = getSingleton(beanName);
        if (sharedInstance != null && args == null) {
            if (logger.isDebugEnabled()) {
                if (isSingletonCurrentlyInCreation(beanName)) {
                    logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                            "' that is not fully initialized yet - a consequence of a circular reference");
                }
                else {
                    logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
                }
            }
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
        }

        else {

            // 因为 Spring 只解决单例模式下得循环依赖，在原型模式下如果存在循环依赖则会抛出异常
            // **关于循环依赖后续会单独出文详细说明**
            if (isPrototypeCurrentlyInCreation(beanName)) {
                throw new BeanCurrentlyInCreationException(beanName);
            }

            // 如果容器中没有找到，则从父类容器中加载
            BeanFactory parentBeanFactory = getParentBeanFactory();
            if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
                String nameToLookup = originalBeanName(name);
                if (parentBeanFactory instanceof AbstractBeanFactory) {
                    return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                            nameToLookup, requiredType, args, typeCheckOnly);
                }
                else if (args != null) {
                    return (T) parentBeanFactory.getBean(nameToLookup, args);
                }
                else {
                    return parentBeanFactory.getBean(nameToLookup, requiredType);
                }
            }

            // 如果不是仅仅做类型检查则是创建bean，这里需要记录
            if (!typeCheckOnly) {
                markBeanAsCreated(beanName);
            }

            try {
                // 从容器中获取 beanName 相应的 GenericBeanDefinition，并将其转换为 RootBeanDefinition
                final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);

                // 检查给定的合并的 BeanDefinition
                checkMergedBeanDefinition(mbd, beanName, args);

                // 处理所依赖的 bean
                String[] dependsOn = mbd.getDependsOn();
                if (dependsOn != null) {
                    for (String dep : dependsOn) {
                        // 若给定的依赖 bean 已经注册为依赖给定的b ean
                        // 循环依赖的情况
                        if (isDependent(beanName, dep)) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                    "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                        }
                        // 缓存依赖调用
                        registerDependentBean(dep, beanName);
                        try {
                            getBean(dep);
                        }
                        catch (NoSuchBeanDefinitionException ex) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                    "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                        }
                    }
                }

                // bean 实例化
                // 单例模式
                if (mbd.isSingleton()) {
                    sharedInstance = getSingleton(beanName, () -> {
                        try {
                            return createBean(beanName, mbd, args);
                        }
                        catch (BeansException ex) {
                            // 显示从单利缓存中删除 bean 实例
                            // 因为单例模式下为了解决循环依赖，可能他已经存在了，所以销毁它
                            destroySingleton(beanName);
                            throw ex;
                        }
                    });
                    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                }

                // 原型模式
                else if (mbd.isPrototype()) {
                    // It's a prototype -> create a new instance.
                    Object prototypeInstance = null;
                    try {
                        beforePrototypeCreation(beanName);
                        prototypeInstance = createBean(beanName, mbd, args);
                    }
                    finally {
                        afterPrototypeCreation(beanName);
                    }
                    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
                }

                else {
                    // 从指定的 scope 下创建 bean
                    String scopeName = mbd.getScope();
                    final Scope scope = this.scopes.get(scopeName);
                    if (scope == null) {
                        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                    }
                    try {
                        Object scopedInstance = scope.get(beanName, () -> {
                            beforePrototypeCreation(beanName);
                            try {
                                return createBean(beanName, mbd, args);
                            }
                            finally {
                                afterPrototypeCreation(beanName);
                            }
                        });
                        bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                    }
                    catch (IllegalStateException ex) {
                        throw new BeanCreationException(beanName,
                                "Scope '" + scopeName + "' is not active for the current thread; consider " +
                                        "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                                ex);
                    }
                }
            }
            catch (BeansException ex) {
                cleanupAfterBeanCreationFailure(beanName);
                throw ex;
            }
        }

        // 检查需要的类型是否符合 bean 的实际类型
        if (requiredType != null && !requiredType.isInstance(bean)) {
            try {
                T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
                if (convertedBean == null) {
                    throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
                }
                return convertedBean;
            }
            catch (TypeMismatchException ex) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Failed to convert bean '" + name + "' to required type '" +
                            ClassUtils.getQualifiedName(requiredType) + "'", ex);
                }
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
        }
        return (T) bean;
    }
```

拆分主要是分为三个部分：

1. 分析从缓存中获取单例 bean，以及对 bean 的实例中获取对象
2. 如果从单例缓存中获取 bean，Spring 是怎么加载的呢？所以第二部分是分析 bean 加载，以及 bean 的依赖处理
3. bean 已经加载了，依赖也处理完毕了，第三部分则分析各个作用域的 bean 初始化过程。

## 1. [单例缓存中获取单例Bean](http://cmsblogs.com/?p=2808)

```java
Object sharedInstance = getSingleton(beanName);
        if (sharedInstance != null && args == null) {
            if (logger.isDebugEnabled()) {
                if (isSingletonCurrentlyInCreation(beanName)) {
                    logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                            "' that is not fully initialized yet - a consequence of a circular reference");
                }
                else {
                    logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
                }
            }
          //如果上面单例缓存中得到了bean，则调用下面方法进行参数进行实例化处理
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
        }
```

### 1. `getSingleton()` 

首先调用`getSingleton()` 从缓存中获取bean，spring对单例模式的bean只会创建一次，后续直接从缓存中取，关键就在这里

```java
public Object getSingleton(String beanName) {
        return getSingleton(beanName, true);
    }

    protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        // 从单例缓冲中加载 bean
        Object singletonObject = this.singletonObjects.get(beanName);

        // 缓存中的 bean 为空，且当前 bean 正在创建
        if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
            // 加锁
            synchronized (this.singletonObjects) {
                // 从 earlySingletonObjects 获取
                singletonObject = this.earlySingletonObjects.get(beanName);
                // earlySingletonObjects 中没有，且允许提前创建
                if (singletonObject == null && allowEarlyReference) {
                    // 从 singletonFactories 中获取对应的 ObjectFactory
                    ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                    // ObjectFactory 不为空，则创建 bean
                    if (singletonFactory != null) {
                        singletonObject = singletonFactory.getObject();
                        this.earlySingletonObjects.put(beanName, singletonObject);
                        this.singletonFactories.remove(beanName);
                    }
                }
            }
        }
        return singletonObject;
    }
```

这个`DefaultSingletonBeanRegistry` 维护了三个map，如下

![KCQnw6.png](/Users/zhangleishuidihuzhu.com/Pictures/wiznote/KCQnw6.png)

这段代码非常简单，首先从 singletonObjects 中获取，若为空且当前 bean 正在创建中，则从 earlySingletonObjects 中获取，若为空且允许提前创建则从 singletonFactories 中获取相应的 ObjectFactory ，若不为空，则调用其 `getObject()`

- singletonObjects ：存放的是单例 bean，对应关系为 `bean name --> bean instance`
- earlySingletonObjects：存放的是早期的 bean，对应关系也是 `bean name --> bean instance`。它与 singletonObjects **区别在于** earlySingletonObjects 中存放的 bean 不一定是完整的，从上面过程中我们可以了解，bean 在创建过程中就已经加入到 earlySingletonObjects 中了，所以当在 bean 的创建过程中就可以通过 `getBean()` 方法获取。这个 Map 也是解决循环依赖的关键所在。
- singletonFactories：存放的是 ObjectFactory，可以理解为创建单例 bean 的 factory，对应关系是 `bean name --> ObjectFactory`

在上面代码中还有一个非常重要的检测方法 `isSingletonCurrentlyInCreation(beanName)` 

```java
protected void beforeSingletonCreation(String beanName) {
		if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}
	}
```

从这段代码中我们可以预测，在 bean 创建过程中都会将其加入到 singletonsCurrentlyInCreation 集合中，具体是在什么时候加的，我们后面分析。

### 2. `getObjectForBeanInstance()` 

，我们再看开篇的代码段，从缓存中获取 bean 后，若其不为 null 且 args 为空，则会调用 `getObjectForBeanInstance()` 处理。为什么会有这么一段呢？

因为我们从缓存中获取的 bean 是最原始的 bean 并不一定使我们最终想要的 bean，怎么办呢？调用 `getObjectForBeanInstance()` 进行处理，该方法的定义为获取给定 bean 实例的对象，该对象要么是 bean 实例本身，要么就是 FactoryBean 创建的对象

```java
protected Object getObjectForBeanInstance(
            Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

        // 若为工厂类引用（name 以 & 开头）
        if (BeanFactoryUtils.isFactoryDereference(name)) {
            // 如果是 NullBean，则直接返回
            if (beanInstance instanceof NullBean) {
                return beanInstance;
            }
            // 如果 beanInstance 不是 FactoryBean 类型，则抛出异常
            if (!(beanInstance instanceof FactoryBean)) {
                throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
            }
        }

        // 到这里我们就有了一个 bean 实例，当然该实例可能是会是是一个正常的 bean 又或者是一个 FactoryBean
        // 如果是 FactoryBean，我我们则创建该 bean
        if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
            return beanInstance;
        }

        // 加载 FactoryBean
        Object object = null;
        // 若 BeanDefinition 为 null，则从缓存中加载
        if (mbd == null) {
            object = getCachedObjectForFactoryBean(beanName);
        }
        // 若 object 依然为空，则可以确认，beanInstance 一定是 FactoryBean
        if (object == null) {
            FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
            //
            if (mbd == null && containsBeanDefinition(beanName)) {
                mbd = getMergedLocalBeanDefinition(beanName);
            }
            // 是否是用户定义的而不是应用程序本身定义的
            boolean synthetic = (mbd != null && mbd.isSynthetic());
            // 核心处理类
            object = getObjectFromFactoryBean(factory, beanName, !synthetic);
        }
        return object;
    }
```

**该方法主要是进行检测工作的，主要如下：**

- 若 name 为工厂相关的（以 & 开头），且 beanInstance 为 NullBean 类型则直接返回，如果 beanInstance 不为 FactoryBean 类型则抛出 BeanIsNotAFactoryException 异常。这里主要是校验 beanInstance 的正确性。
- 如果 beanInstance 不为 FactoryBean 类型或者 name 也不是与工厂相关的，则直接返回。这里主要是对非 FactoryBean 类型处理。
- 如果 BeanDefinition 为空，则从 factoryBeanObjectCache 中加载，如果还是空，则可以断定 beanInstance 一定是 FactoryBean 类型，则委托 `getObjectFromFactoryBean()` 方法处理

> 从这里可以看出，该方法主要返回给定的bean实例对象，当然该实例对象为非 FactoryBean 类型，对于 FactoryBean 类型的 bean，则是委托 `getObjectFromFactoryBean()` 从 FactoryBean 获取 bean 实例对象。

### 3. `getObjectFactoryBean()` 

```java
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
        // 为单例模式且缓存中存在
        if (factory.isSingleton() && containsSingleton(beanName)) {

            synchronized (getSingletonMutex()) {
                // 从缓存中获取指定的 factoryBean
                Object object = this.factoryBeanObjectCache.get(beanName);

                if (object == null) {
                    // 为空，则从 FactoryBean 中获取对象
                    object = doGetObjectFromFactoryBean(factory, beanName);

                    // 从缓存中获取
                    Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
                    // **我实在是不明白这里这么做的原因，这里是干嘛？？？**
                    if (alreadyThere != null) {
                        object = alreadyThere;
                    }
                    else {
                        // 需要后续处理
                        if (shouldPostProcess) {
                            // 若该 bean 处于创建中，则返回非处理对象，而不是存储它
                            if (isSingletonCurrentlyInCreation(beanName)) {
                                return object;
                            }
                            // 前置处理
                            beforeSingletonCreation(beanName);
                            try {
                                // 对从 FactoryBean 获取的对象进行后处理
                                // 生成的对象将暴露给bean引用
                                object = postProcessObjectFromFactoryBean(object, beanName);
                            }
                            catch (Throwable ex) {
                                throw new BeanCreationException(beanName,
                                        "Post-processing of FactoryBean's singleton object failed", ex);
                            }
                            finally {
                                // 后置处理
                                afterSingletonCreation(beanName);
                            }
                        }
                        // 缓存
                        if (containsSingleton(beanName)) {
                            this.factoryBeanObjectCache.put(beanName, object);
                        }
                    }
                }
                return object;
            }
        }
        else {
            // 非单例
            Object object = doGetObjectFromFactoryBean(factory, beanName);
            if (shouldPostProcess) {
                try {
                    object = postProcessObjectFromFactoryBean(object, beanName);
                }
                catch (Throwable ex) {
                    throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
                }
            }
            return object;
        }
    }
```

主要流程如下：

- 若为单例且单例 bean 缓存中存在 beanName，则进行后续处理（跳转到下一步），否则则从 FactoryBean 中获取 bean 实例对象，如果接受后置处理，则调用 `postProcessObjectFromFactoryBean()` 进行后置处理。
- 首先获取锁（其实我们在前面篇幅中发现了大量的同步锁，锁住的对象都是 this.singletonObjects， 主要是因为在单例模式中必须要保证全局唯一），然后从 factoryBeanObjectCache 缓存中获取实例对象 object，若 object 为空，则调用 `doGetObjectFromFactoryBean()` 方法从 FactoryBean 获取对象，其实内部就是调用 `FactoryBean.getObject()`。
- 如果需要后续处理，则进行进一步处理，步骤如下：
  - 若该 bean 处于创建中（isSingletonCurrentlyInCreation），则返回非处理对象，而不是存储它
  - 调用 `beforeSingletonCreation()` 进行创建之前的处理。默认实现将该 bean 标志为当前创建的。
  - 调用 `postProcessObjectFromFactoryBean()` 对从 FactoryBean 获取的 bean 实例对象进行后置处理，默认实现是按照原样直接返回，具体实现是在 AbstractAutowireCapableBeanFactory 中实现的，当然子类也可以重写它，比如应用后置处理
  - 调用 `afterSingletonCreation()` 进行创建 bean 之后的处理，默认实现是将该 bean 标记为不再在创建中。
- 最后加入到 FactoryBeans 缓存中。

该方法应该就是创建 bean 实例对象中的核心方法之一了。这里我们关注三个方法：`beforeSingletonCreation()` 、 `afterSingletonCreation()` 、 `postProcessObjectFromFactoryBean()`。可能有小伙伴觉得前面两个方法不是很重要，LZ 可以肯定告诉你，这两方法是非常重要的操作，因为**他们记录着 bean 的加载状态，是检测当前 bean 是否处于创建中的关键之处，对解决 bean 循环依赖起着关键作用**。before 方法用于标志当前 bean 处于创建中，after 则是移除。

## 2. [parentBeanFactory 与依赖处理](http://cmsblogs.com/?p=2810)

>  如果从单例缓存中没有获取到单例 bean，则说明两种情况：
>
> 1. 该 bean 的 scope 不是 singleton
> 2. 该 bean 的 scope 是 singleton ,但是没有初始化完成

针对这两种情况 Spring 是如何处理的呢？统一加载并完成初始化！这部分内容的篇幅较长，拆分为两部分，第一部分主要是一些检测、parentBeanFactory 以及依赖处理，第二部分则是各个 scope 的初始化。

```java
 if (isPrototypeCurrentlyInCreation(beanName)) {
                throw new BeanCurrentlyInCreationException(beanName);
            }

            // Check if bean definition exists in this factory.
            BeanFactory parentBeanFactory = getParentBeanFactory();
            if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
                // Not found -> check parent.
                String nameToLookup = originalBeanName(name);
                if (parentBeanFactory instanceof AbstractBeanFactory) {
                    return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                            nameToLookup, requiredType, args, typeCheckOnly);
                }
                else if (args != null) {
                    // Delegation to parent with explicit args.
                    return (T) parentBeanFactory.getBean(nameToLookup, args);
                }
                else {
                    // No args -> delegate to standard getBean method.
                    return parentBeanFactory.getBean(nameToLookup, requiredType);
                }
            }
            if (!typeCheckOnly) {
                markBeanAsCreated(beanName);
            }

            try {
                final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
                checkMergedBeanDefinition(mbd, beanName, args);

                // Guarantee initialization of beans that the current bean depends on.
                String[] dependsOn = mbd.getDependsOn();
                if (dependsOn != null) {
                    for (String dep : dependsOn) {
                        if (isDependent(beanName, dep)) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                    "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                        }
                        registerDependentBean(dep, beanName);
                        try {
                            getBean(dep);
                        }
                        catch (NoSuchBeanDefinitionException ex) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                    "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                        }
                    }
                }
            }
            // 省略很多代码
```

这段代码主要处理如下几个部分：

1. 检测。若当前 bean 在创建，则抛出 BeanCurrentlyInCreationException 异常。

   > **检测** 在前面就提过，Spring 只解决单例模式下的循环依赖，对于原型模式的循环依赖则是抛出 BeanCurrentlyInCreationException 异常，所以首先检查该 beanName 是否处于原型模式下的循环依赖.
   >
   > 其实检测逻辑和单例模式一样，一个“集合”存放着正在创建的 bean，从该集合中进行判断即可，只不过单例模式的“集合”为 Set ，而原型模式的则是 ThreadLocal，prototypesCurrentlyInCreation 定义如下
   >
   > `private final ThreadLocal<Object> prototypesCurrentlyInCreation = new NamedThreadLocal<>("Prototype beans currently in creation");`

2. 如果 beanDefinitionMap 中不存在 beanName 的 BeanDefinition（即在 Spring bean 初始化过程中没有加载），则尝试从 parentBeanFactory 中加载。

3. 判断是否为类型检查。

4. 从 mergedBeanDefinitions 中获取 beanName 对应的 RootBeanDefinition，如果这个 BeanDefinition 是子 Bean 的话，则会合并父类的相关属性。

5. 依赖处理。

   > 其实将就是该映射关系保存到两个集合中：dependentBeanMap、dependenciesForBeanMap。 最后调用 `getBean()` 实例化依赖 bean。 至此，加载 bean 的第二个部分也分析完毕了，下篇开始分析第三个部分：各大作用域 bean 的处理

## 3. [各 scope 的 bean 创建](http://cmsblogs.com/?p=2839)

紧接上面的代码，开始对scope进行校验

```java
if (mbd.isSingleton()) {
                    sharedInstance = getSingleton(beanName, () -> {
                        try {
                            return createBean(beanName, mbd, args);
                        }
                        catch (BeansException ex) {
                            destroySingleton(beanName);
                            throw ex;
                        }
                    });
                    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                }
```

上面分析了从缓存中获取单例模式的bean，但如果缓存不存在?  则由`getSingleton()` 实现

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(beanName, "Bean name must not be null");

        // 全局加锁
        synchronized (this.singletonObjects) {
            // 从缓存中检查一遍
            // 因为 singleton 模式其实就是复用已经创建的 bean 所以这步骤必须检查
            Object singletonObject = this.singletonObjects.get(beanName);
            //  为空，开始加载过程
            if (singletonObject == null) {
                // 省略 部分代码

                // 加载前置处理
                beforeSingletonCreation(beanName);
                boolean newSingleton = false;
                // 省略代码
                try {
                    // 初始化 bean
                    // 这个过程其实是调用 createBean() 方法
                    singletonObject = singletonFactory.getObject();
                    newSingleton = true;
                }
                // 省略 catch 部分
                }
                finally {
                    // 后置处理
                    afterSingletonCreation(beanName);
                }
                // 加入缓存中
                if (newSingleton) {
                    addSingleton(beanName, singletonObject);
                }
            }
            // 直接返回
            return singletonObject;
        }
    }
```

其实这个过程并没有真正创建 bean，仅仅只是做了一部分准备和预处理步骤，真正获取单例 bean 的方法其实是由 `singletonFactory.getObject()`

1. 再次检查缓存是否已经加载过，如果已经加载了则直接返回，否则开始加载过程。
2. 调用 `beforeSingletonCreation()` 记录加载单例 bean 之前的加载状态，即前置处理。
3. 调用参数传递的 ObjectFactory 的 `getObject()` 实例化 bean。
4. 调用 `afterSingletonCreation()` 进行加载单例后的后置处理。
5. 将结果记录并加入值缓存中，同时删除加载 bean 过程中所记录的一些辅助状态。

这里before和after已经说过，看看`addSingleton()` 

```java
protected void addSingleton(String beanName, Object singletonObject) {
		synchronized (this.singletonObjects) {
			this.singletonObjects.put(beanName, singletonObject);
			this.singletonFactories.remove(beanName);
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.add(beanName);
		}
	}
```

一个put，两个remove，一个add。

- singletonObjects: 单例bean的缓存
- singletonFactories ： 单例bean Factory 的缓存
- earlySingletonObjects：早起创建的单例bean的缓存
- registeredSingletons ： 已经注册的单例缓存

加载了单例bean后，调用`getObjectForBeanInstance()` 从bean实例中获取对象

**原型模式**

```java
else if (mbd.isPrototype()) {
                    Object prototypeInstance = null;
                    try {
                        beforePrototypeCreation(beanName);
                        prototypeInstance = createBean(beanName, mbd, args);
                    }
                    finally {
                        afterPrototypeCreation(beanName);
                    }
                    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
                }
```

原型模式的初始化过程很简单：直接创建一个新的实例就可以了。过程如下：

1. 调用 `beforeSingletonCreation()` 记录加载原型模式 bean 之前的加载状态，即前置处理。
2. 调用 `createBean()` 创建一个 bean 实例对象。
3. 调用 `afterSingletonCreation()` 进行加载原型模式 bean 后的后置处理。
4. 调用 `getObjectForBeanInstance()` 从 bean 实例中获取对象。

其他scope，自己看看

对于上面三个模块，其中最重要的有两个方法，一个是 `createBean()`、一个是 `getObjectForBeanInstance()`。这两个方法在上面三个模块都有调用，`createBean()` 后续详细说明，

`getObjectForBeanInstance()` 在博客 [【死磕 Spring】----- 加载 bean 之 缓存中获取单例 bean](http://cmsblogs.com/?p=2808) 中有详细讲解，这里再次阐述下（此段内容来自《Spring 源码深度解析》）：这个方法主要是验证以下我们得到的 bean 的正确性，其实就是检测当前 bean 是否是 FactoryBean 类型的 bean，如果是，那么需要调用该 bean 对应的 FactoryBean 实例的 `getObject()` 作为返回值。

无论是从缓存中获得到的 bean 还是通过不同的 scope 策略加载的 bean 都只是最原始的 bean 状态，并不一定就是我们最终想要的 bean。举个例子，假如我们需要对工厂 bean 进行处理，那么这里得到的其实是工厂 bean 的初始状态，但是我们真正需要的是工厂 bean 中定义 factory-method 方法中返回的 bean，而 `getObjectForBeanInstance()` 就是完成这个工作的。 至此，Spring 加载 bean 的三个部分（LZ自己划分的）已经分析完毕了。

## 4. Bean实例化

> 这里我们来看看  createBean方法

```java
protected abstract Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
            throws BeanCreationException;
```

该方法定义在 AbstractBeanFactory 中。其含义是根据给定的 BeanDefinition 和 args实例化一个 bean 对象，如果该 BeanDefinition 存在父类，则该 BeanDefinition 已经合并了父类的属性。所有 Bean 实例的创建都会委托给该方法实现。 方法接受三个参数：

- beanName：bean 的名字
- mbd：已经合并了父类属性的（如果有的话）BeanDefinition
- args：用于构造函数或者工厂方法创建 bean 实例对象的参数

该方法默认实现在，类 `AbstractAutowireCapableBeanFactory`  中实现

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
            throws BeanCreationException {

        if (logger.isDebugEnabled()) {
            logger.debug("Creating instance of bean '" + beanName + "'");
        }
        RootBeanDefinition mbdToUse = mbd;

        // 确保此时的 bean 已经被解析了
        // 如果获取的class 属性不为null，则克隆该 BeanDefinition
        // 主要是因为该动态解析的 class 无法保存到到共享的 BeanDefinition
        Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
        if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
            mbdToUse = new RootBeanDefinition(mbd);
            mbdToUse.setBeanClass(resolvedClass);
        }

        try {
            // 验证和准备覆盖方法
            mbdToUse.prepareMethodOverrides();
        }
        catch (BeanDefinitionValidationException ex) {
            throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
                    beanName, "Validation of method overrides failed", ex);
        }

        try {
            // 给 BeanPostProcessors 一个机会用来返回一个代理类而不是真正的类实例
            // AOP 的功能就是基于这个地方
            Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
            if (bean != null) {
                return bean;
            }
        }
        catch (Throwable ex) {
            throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
                    "BeanPostProcessor before instantiation of bean failed", ex);
        }

        try {
            // 执行真正创建 bean 的过程
            Object beanInstance = doCreateBean(beanName, mbdToUse, args);
            if (logger.isDebugEnabled()) {
                logger.debug("Finished creating instance of bean '" + beanName + "'");
            }
            return beanInstance;
        }
        catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanCreationException(
                    mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
        }
    }
```

过程如下：

- 解析指定 BeanDefinition 的 class
- 处理 override 属性
- 实例化的前置处理
- 创建 bean

**解析指定BeanDefinition的Class**

```java
Class<?> resolvedClass = resolveBeanClass(mbd, beanName)
```

这个方法主要是解析 bean definition 的 class 类，并将已经解析的 Class 存储在 bean definition 中以供后面使用。如果解析的 class 不为空，则会将该 BeanDefinition 进行克隆至 mbdToUse，这样做的主要目的是因为动态解析的 class 是无法保存到共享的 BeanDefinition 中。 **处理 override 属性** 

大家还记得 lookup-method 和 replace-method 这两个配置功能？在博客 [【死磕 Spring】----- IOC 之解析Bean：解析 bean 标签（三）](http://cmsblogs.com/?p=2846) 中已经详细分析了这两个标签的用法和解析过程，知道解析过程其实就是将这两个配置存放在 BeanDefinition 中的 methodOverrides 属性中，我们知道在 bean 实例化的过程中如果检测到存在 methodOverrides，则会动态地位为当前 bean 生成代理并使用对应的拦截器为 bean 做增强处理。具体的实现我们后续分析

现在先看 `mbdToUse.prepareMethodOverrides()` 都干了些什么事，如下

```java
 public void prepareMethodOverrides() throws BeanDefinitionValidationException {
        if (hasMethodOverrides()) {
            Set<MethodOverride> overrides = getMethodOverrides().getOverrides();
            synchronized (overrides) {
                for (MethodOverride mo : overrides) {
                    prepareMethodOverride(mo);
                }
            }
        }
    }
```

如果存在 methodOverrides 则获取所有的 override method ，然后通过迭代的方法一次调用 `prepareMethodOverride()` ，如下：

```java
protected void prepareMethodOverride(MethodOverride mo) throws BeanDefinitionValidationException {
        int count = ClassUtils.getMethodCountForName(getBeanClass(), mo.getMethodName());
        if (count == 0) {
            throw new BeanDefinitionValidationException(
                    "Invalid method override: no method with name '" + mo.getMethodName() +
                    "' on class [" + getBeanClassName() + "]");
        }
        else if (count == 1) {
            mo.setOverloaded(false);
        }
    }
```

根据**方法名称**从 class 中获取该方法名的个数，如果为 0 则抛出异常，如果 为 1 则设置该重载方法没有被重载。**若一个类中存在多个重载方法**，则在方法调用的时候还需要**根据参数类型**来判断到底重载的是哪个方法。在设置重载的时候其实这里做了一个小小优化，那就是当 `count == 1` 时，设置 `overloaded = false`，这样表示该方法没有重载，这样在后续调用的时候便可以直接找到方法而不需要进行方法参数的校验。 

诚然，其实 `mbdToUse.prepareMethodOverrides()` 并没有做什么实质性的工作，只是对 methodOverrides 属性做了一些简单的校验而已。 **实例化的前置处理** `resolveBeforeInstantiation()` 的作用是给 BeanPostProcessors 后置处理器返回一个代理对象的机会，其实在调用该方法之前 Spring 一直都没有创建 bean ，那么这里返回一个 bean 的代理类有什么作用呢？作用体现在后面的 `if` 判断：

```java
if (bean != null) {
    return bean;
}
```

***如果代理对象不为空，则直接返回代理对象，这一步骤有非常重要的作用，Spring 后续实现 AOP 就是基于这个地方判断的。***

```java
 protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
        Object bean = null;
        if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
            if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
                Class<?> targetType = determineTargetType(beanName, mbd);
                if (targetType != null) {
                    bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                    if (bean != null) {
                        bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                    }
                }
            }
            mbd.beforeInstantiationResolved = (bean != null);
        }
        return bean;
    }
```

这个方法核心就在于 `applyBeanPostProcessorsBeforeInstantiation()` 和 `applyBeanPostProcessorsAfterInitialization()` 两个方法，before 为实例化前的后处理器应用，after 为实例化后的后处理器应用，由于本文的主题是创建 bean，关于 Bean 的增强处理后续 LZ 会单独出博文来做详细说明。 **创建 bean** 如果没有<u>代理对象</u>，就只能走常规的路线进行 bean 的创建了，该过程有 `doCreateBean()` 实现，如下：

```xml
-- 代码太长，省略，后面会详细介绍
```

整体的思路：

1. 如果是单例模式，则清除 factoryBeanInstanceCache 缓存，同时返回 BeanWrapper 实例对象，当然如果存在。
2. 如果缓存中没有 BeanWrapper 或者不是单例模式，则调用 `createBeanInstance()` 实例化 bean，主要是将 BeanDefinition 转换为 BeanWrapper

\- MergedBeanDefinitionPostProcessor 的应用 - 单例模式的循环依赖处理 - 调用 `populateBean()``initializeBean()``doCreateBean()`

- `createBeanInstance()` 实例化 bean
- `populateBean()` 属性填充
- 循环依赖的处理
- `initializeBean()` 初始化 bean

#### 1. 创建bean的第一个步骤：实例化bean

> 对应的方法是  createBeanInstance() 

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
        // 解析 bean，将 bean 类名解析为 class 引用
        Class<?> beanClass = resolveBeanClass(mbd, beanName);

        if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
        }

        // 如果存在 Supplier 回调，则使用给定的回调方法初始化策略
        Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
        if (instanceSupplier != null) {
            return obtainFromSupplier(instanceSupplier, beanName);
        }

        // 如果工厂方法不为空，则使用工厂方法初始化策略
        if (mbd.getFactoryMethodName() != null)  {
            return instantiateUsingFactoryMethod(beanName, mbd, args);
        }

        boolean resolved = false;
        boolean autowireNecessary = false;
        if (args == null) {
            // constructorArgumentLock 构造函数的常用锁
            synchronized (mbd.constructorArgumentLock) {
                // 如果已缓存的解析的构造函数或者工厂方法不为空，则可以利用构造函数解析
                // 因为需要根据参数确认到底使用哪个构造函数，该过程比较消耗性能，所有采用缓存机制
                if (mbd.resolvedConstructorOrFactoryMethod != null) {
                    resolved = true;
                    autowireNecessary = mbd.constructorArgumentsResolved;
                }
            }
        }
        // 已经解析好了，直接注入即可
        if (resolved) {
            // 自动注入，调用构造函数自动注入
            if (autowireNecessary) {
                return autowireConstructor(beanName, mbd, null, null);
            }
            else {
                // 使用默认构造函数构造
                return instantiateBean(beanName, mbd);
            }
        }

        // 确定解析的构造函数
        // 主要是检查已经注册的 SmartInstantiationAwareBeanPostProcessor
        Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
        if (ctors != null ||
                mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
                mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
            // 构造函数自动注入
            return autowireConstructor(beanName, mbd, ctors, args);
        }

        //使用默认构造函数注入
        return instantiateBean(beanName, mbd);
    }
```

实例化 bean 是一个复杂的过程，其主要的逻辑为：

- 如果存在 Supplier 回调，则调用 `obtainFromSupplier()` 进行初始化
- 如果存在工厂方法，则使用工厂方法进行初始化
- 首先判断缓存，如果缓存中存在，即已经解析过了，则直接使用已经解析了的，根据 constructorArgumentsResolved 参数来判断是使用构造函数自动注入还是默认构造函数
- 如果缓存中没有，则需要先确定到底使用哪个构造函数来完成解析工作，因为一个类有多个构造函数，每个构造函数都有不同的构造参数，所以需要根据参数来锁定构造函数并完成初始化，如果存在参数则使用相应的带有参数的构造函数，否则使用默认构造函数

##### 1. obtainFromSupplier()

```java
Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
if (instanceSupplier != null) {
    return obtainFromSupplier(instanceSupplier, beanName);
}
```

先从BeanDefinition中获取Supplier，如果不为空，调用 `obtainFromSupplier()` 。那么 Supplier 是什么呢？在这之前也没有提到过这个字段。

```java
public interface Supplier<T> {
    T get();
}
```

Supplier 接口仅有一个功能性的 `get()`，该方法会返回一个 T 类型的对象，有点儿类似工厂方法。这个接口有什么作用？用于指定创建 bean 的回调，如果我们设置了这样的回调，那么其他的构造器或者工厂方法都会没有用。在什么设置该参数呢？Spring 提供了相应的 `setter` 方法，如下：

```java
 public void setInstanceSupplier(@Nullable Supplier<?> instanceSupplier) {
        this.instanceSupplier = instanceSupplier;
    }
```

在**构造 BeanDefinition 的时候设置了该值**，如下（以 RootBeanDefinition 为例）：

```java
    public <T> RootBeanDefinition(@Nullable Class<T> beanClass, String scope, @Nullable Supplier<T> instanceSupplier) {
        super();
        setBeanClass(beanClass);
        setScope(scope);
        setInstanceSupplier(instanceSupplier);
    }
```

如果设置了 instanceSupplier 则调用 `obtainFromSupplier()`

```java
protected BeanWrapper obtainFromSupplier(Supplier<?> instanceSupplier, String beanName) {
        String outerBean = this.currentlyCreatedBean.get();
        this.currentlyCreatedBean.set(beanName);
        Object instance;
        try {
          // 调用 Supplier 的 get()，返回一个对象
            instance = instanceSupplier.get();
        }
        finally {
            if (outerBean != null) {
                this.currentlyCreatedBean.set(outerBean);
            }
            else {
                this.currentlyCreatedBean.remove();
            }
        }
        // 根据对象构造 BeanWrapper 对象
        BeanWrapper bw = new BeanWrapperImpl(instance);
        // 初始化 BeanWrapper
        initBeanWrapper(bw);
        return bw;
    }
```

代码很简单，调用 调用 Supplier 的 `get()` 方法，获得一个 bean 实例对象，然后根据该实例对象构造一个 BeanWrapper 对象 bw，最后初始化该对象。有关于 BeanWrapper 后面专门出文讲解。

##### 2. instantiateUsingFactoryMethod()

如果存在工厂方法，则调用 `instantiateUsingFactoryMethod()` 完成 bean 的初始化工作（方法实现比较长，细节比较复杂，各位就硬着头皮看吧）。

```java
  protected BeanWrapper instantiateUsingFactoryMethod(
            String beanName, RootBeanDefinition mbd, @Nullable Object[] explicitArgs) {
        return new ConstructorResolver(this).instantiateUsingFactoryMethod(beanName, mbd, explicitArgs);
    }
```

构造一个 ConstructorResolver 对象，然后调用其 `instantiateUsingFactoryMethod()` 方法。ConstructorResolver 是构造方法或者工厂类初始化 bean 的委托类。

> 这个方法实在是长，就不贴了 http://cmsblogs.com/?p=2848

`instantiateUsingFactoryMethod()` 方法体实在是太大了，处理细节感觉很复杂，LZ是硬着头皮看完的，中间断断续续的。吐槽这里的代码风格，完全不符合我们前面看的 Spring 代码风格。Spring 的一贯做法是将一个复杂逻辑进行拆分，分为多个细小的模块进行嵌套，每个模块负责一部分功能，模块与模块之间层层嵌套，上一层一般都是对下一层的总结和概括，这样就会使得每一层的逻辑变得清晰易懂。

 回归到上面的方法体，虽然代码体量大，但是总体我们还是可看清楚这个方法要做的事情。一句话概括就是：确定工厂对象，然后获取构造函数和构造参数，最后调用 InstantiationStrategy 对象的 `instantiate()` 来创建 bean 实例。下面我们就这个句概括的话进行拆分并详细说明。 

**确定工厂对象** 首先获取工厂方法名，若工厂方法名不为空，则调用 `beanFactory.getBean()` 获取工厂对象，若为空，则可能为一个静态工厂，对于静态工厂则必须提供工厂类的全类名，同时设置 `factoryBean = null` 

**构造参数确认** 工厂对象确定后，则是确认构造参数。构造参数的确认主要分为三种情况：explicitArgs 参数、缓存中获取、配置文件中解析。 

**explicitArgs 参数** explicitArgs 参数是我们调用 `getBean()` 时传递景来，一般该参数，该参数就是用于初始化 bean 时所传递的参数，如果该参数不为空，则可以确定构造函数的参数就是它了。

**缓存中获取** 在该方法的最后，我们会发现这样一段代码：`argsHolderToUse.storeCache(mbd, factoryMethodToUse)` ，这段代码主要是将构造函数、构造参数保存到缓存中，如下：

