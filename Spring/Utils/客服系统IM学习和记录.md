# 客服系统IM学习和记录

> 作为一个系统的核心部分，IM是其中至关重要的部分，这篇主要记述IM部分的相关东西，学习如何处理问题，技术选型，系统迭代，源码阅读。

## 1. 客服IM选型介绍和建立连接

【客服架构图片】

本次主要讲，如何配置grpc，如何建立连接，如何获取连接

### 框架的选择: Socket.io

> [Socket.io提供了基于事件的实时双向通讯](https://www.jianshu.com/p/4e80b931cdea)
>
> 2011年IETF标准化了WebSocket：一种基于TCP套接字进行首发数据的协议
>
> Socket.io将数据传输部分独立出来形成engine.io，engine.io对WebSocket和AJAX轮询进行了封装，形成了一套API，屏蔽了细节差异和兼容性问题，实现了跨浏览器/跨设备进行双向数据通信。

Socket.iol是一个跨浏览器支持WebSocket的实时通讯的JS   [Socket.io官网](https://socket.io/docs/)

### Socket.io的基本应用

> socket.io提供了基于事件的实时双向通讯，**它同时提供了服务端和客户端的API**。

**服务端**

1. 配置

服务端socket.io必须绑定一个`http.Server`实例，因为WebSocket协议是构建在HTTP协议之上的，所以在创建WebSocket服务时需调用HTTP模块并调用其下`createServer()`方法，将生成的server作为参数传入socket.io。

我们的代码是在  imconfig.java这个类中进行了绑定

![euta5j.png](https://s2.ax1x.com/2019/07/26/euta5j.png)

这里绑定了端口，设置长连接，监听器，并且创建了server

2. 建立连接



