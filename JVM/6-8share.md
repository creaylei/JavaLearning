https://mp.weixin.qq.com/s/yBrCiaZV5CbEurppYXpDvA

实现一个Mybitis插件

先来看一个分页插件 PageHelper 的示例

![image-20200607175330927](https://tva1.sinaimg.cn/large/007S8ZIlly1gfjvibhkevj31zq0dm433.jpg)

这里有三个要点：

1. 实现了mybitis的拦截器
2. @Intercepts注解
3. @Signature注解



