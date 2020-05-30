# Ribbon源码解析

> Ribbon是Netflix公司开源的一个负载均衡的项目，它是一个客户端负载均衡器，运行在客户端上。 Feign已经默认慎用了Ribbon

- 负载均衡
- 容错
- 多协议(HTTP, TCP, UDP) 支持异步和反应模型
- 缓存和批处理



## 使用:RestTemplate和Ribbon的结合

```java
@Configuration
public class RibbonConfig {
    @Bean
    @LoadBalanced
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

消费另外一个的服务的接口，差不多这样：

```java
@Service
public class RibbonService {
    @Autowired
    RestTemplate restTemplate;
    public String hi(String name) {
        return restTemplate.getForObject("http://eureka-client/hi?name="+name,String.class);
    }
}
```

这样是用RestTemplate可以通过ribbon进行负载均衡。 那么ribbon是怎么工作的， 即@LoadBalanced是什么原理，下面就来看一看。

## 源码：深入理解Ribbon

### 1. LoadBalancerClient 组件

![类图](/Users/zhangleishuidihuzhu.com/Pictures/wiznote/LoadBalancerClient.png)



`LoadBalancerClient` 是一个接口，继承了 `ServiceInstanceChooser` , 实现类是 `RibbonLoadBalancerClient` 来分别看一下：

1. `ServiceInstanceChooser`

```java
public interface ServiceInstanceChooser {
    ServiceInstance choose(String serviceId);
}
```

只有一个获取应用实例的方法。 根据服务id来获取

2. `LoadBalancerClient`

```java
public interface LoadBalancerClient extends ServiceInstanceChooser {
    <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;

    <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException;

    URI reconstructURI(ServiceInstance instance, URI original);
}
```

执行execute，和重构Uri的`reconstructURI`

3. 再来看下实现类`RibbonLoadBalancerClient`

```java
public class RibbonLoadBalancerClient implements LoadBalancerClient {

...//省略代码

  //实现 choose方法，进行负载均衡
@Override
	public ServiceInstance choose(String serviceId) {
		Server server = getServer(serviceId);    //获取服务实例，最后交由loadBalancer选择服务实例
		if (server == null) {
			return null;
		}
		return new RibbonServer(serviceId, server, isSecure(server, serviceId),
				serverIntrospector(serviceId).getMetadata(server));
	}



protected Server getServer(String serviceId) {
		return getServer(getLoadBalancer(serviceId));
   }


protected Server getServer(ILoadBalancer loadBalancer) {
		if (loadBalancer == null) {
			return null;
		}
		return loadBalancer.chooseServer("default"); // TODO: better handling of key
	}


protected ILoadBalancer getLoadBalancer(String serviceId) {
		return this.clientFactory.getLoadBalancer(serviceId);
	}
	
	...//省略代码
```

在RibbonLoadBalancerClient的源码中，其中choose()方法是选择具体服务实例的一个方法。该方法通过getServer()方法去获取实例，经过源码跟踪，最终交给了ILoadBalancer类去选择服务实例。

```java
public interface ILoadBalancer {

    public void addServers(List<Server> newServers);
    public Server chooseServer(Object key);
    public void markServerDown(Server server);
    public List<Server> getReachableServers();
    public List<Server> getAllServers();
}
```

`ILoadBalancer` 定义了实现软件负载均衡的接口，它需要一组可供选择的服务注册列表信息，以及根据特定方法去选择服务。      

其中，addServers()方法是添加一个Server集合；chooseServer()方法是根据key去获取Server；markServerDown()方法用来标记某个服务下线；getReachableServers()获取可用的Server集合；getAllServers()获取所有的Server集合。

也就是说负载均衡的核心就是它。 `ILoadBalancer` 

---

### 2. DynamicServerListLoadBalancer

那先来看看`ILoadBalancer`的类图，实现类有哪些

![image-20191119155900852](/Users/zhangleishuidihuzhu.com/Pictures/wiznote/image-20191119155900852.png)

1. 先看看 `AbstractLoadBalancer` 

```java
public abstract class AbstractLoadBalancer implements ILoadBalancer {
    
    public enum ServerGroup{
        ALL,
        STATUS_UP,
        STATUS_NOT_UP        
    }
        
    public Server chooseServer() {
    	return chooseServer(null);
    }
    public abstract List<Server> getServerList(ServerGroup serverGroup);
    
    public abstract LoadBalancerStats getLoadBalancerStats();    
}
```

它有一个枚举，所有服务，可用的服务 up , 不可以的服务 status_not_up 。  `getLoadBalancerStats` 获取所有LoadBalancer的数据，`getServerList` 获取所有服务

2. 再往下走 看下他是实现类，`BaseLoadBalancer` 类

```java
public class BaseLoadBalancer extends AbstractLoadBalancer implements
        PrimeConnections.PrimeConnectionListener, IClientConfigAware {

    private final static IRule DEFAULT_RULE = new RoundRobinRule();
    private final static SerialPingStrategy DEFAULT_PING_STRATEGY = new SerialPingStrategy();
    private static final String DEFAULT_NAME = "default";
    private static final String PREFIX = "LoadBalancer_";

    protected IRule rule = DEFAULT_RULE;

    protected IPing ping = null;
    
    private IClientConfig config;
}
```

他配置了  IRule   、  IPing 、IClientConfig

3. 再往下`DynamicServerListLoadBalancer` 

```java
public class DynamicServerListLoadBalancer<T extends Server> extends BaseLoadBalancer {

    volatile ServerList<T> serverListImpl;

    volatile ServerListFilter<T> filter;
```

加上 ServerList   和 ServerListFilter   和 ILoadBalancer， 集齐所有6个功能

- **IClientConfig** ribbonClientConfig: DefaultClientConfigImpl配置
- **IRule** ribbonRule: RoundRobinRule 路由策略
- **IPing** ribbonPing: DummyPing    会通过ping来判断server是否可用
- **ServerList** ribbonServerList: ConfigurationBasedServerList
- **ServerListFilter** ribbonServerListFilter: ZonePreferenceServerListFilter
- **ILoadBalancer** ribbonLoadBalancer: ZoneAwareLoadBalancer

下面来详细介绍下这几个配置。

#### 1. IClientConfig 

用于对客户端或者负载均衡的配置，它的默认实现类为DefaultClientConfigImpl。  可以看看

#### 2. IRule用于复杂均衡的策略

```java
public interface IRule{
    public Server choose(Object key);     //通过key获取 server
     
    public void setLoadBalancer(ILoadBalancer lb);   //设置LoadBalancer
    
    public ILoadBalancer getLoadBalancer();       //获取LoadBalancer
}
```

它的实现类如下

![image-20191119163249535](/Users/zhangleishuidihuzhu.com/Pictures/wiznote/image-20191119163249535.png)

- BestAvailableRule 选择最小请求数
- ClientConfigEnabledRoundRobinRule 轮询
- RandomRule 随机选择一个server
- RoundRobinRule 轮询选择server
- RetryRule 根据轮询的方式重试
- WeightedResponseTimeRule 根据响应时间去分配一个weight ，weight越低，被选择的可能性就越低
- ZoneAvoidanceRule 根据server的zone区域和可用性来轮询选择

#### 3. IPing 用来ping Server

IPing是用来向Server发生"ping", 来判断该server是否有响应，从而判断该server是否可用。 源码如下

```java
public interface IPing {
    public boolean isAlive(Server server);        //是否存活的方法
}
```

![image-20191119171829941](/Users/zhangleishuidihuzhu.com/Pictures/wiznote/image-20191119171829941.png)

- PingUrl 真实的去ping 某个url，判断其是否alive
- PingConstant 固定返回某服务是否可用，默认返回true，即可用
- NoOpPing 不去ping,直接返回true,即可用。
- DummyPing 直接返回true，并实现了initWithNiwsConfig方法。
- NIWSDiscoveryPing，根据DiscoveryEnabledServer的InstanceInfo的InstanceStatus去判断，如果为InstanceStatus.UP，则为可用，否则不可用。

#### 4. ServerList

是定义获取所有的server的注册列表信息的接口，它的代码如下

```java
public interface ServerList<T extends Server> {

    public List<T> getInitialListOfServers();
    public List<T> getUpdatedListOfServers();   
}
```

#### 5. ServerListFilter

定义了 可以根据配置，或者根据特性动态符合条件的 server列表的方法。 代码如下:

```
public interface ServerListFilter<T extends Server> {

    public List<T> getFilteredListOfServers(List<T> servers);

}
```