Netty实战

> 2021-0702

全书分为四部分

1. Netty概念和体系
2. 编解码器
3. 网络协议
4. 案例研究



[toc]

## 第一部分：Netty概念及体系结构

### 第一章 Netty-异步和事件驱动

#### 1. 传统的`JavaSocket`编程

```java
ServerSocket serverSocket = new ServerSocket(portNumber); ← -- 创􏰕一个新的ServerSocket，用以监听指定 

Socket clientSocket = serverSocket.accept(); ← -- ❶ 对accept()方法􏳛的􏱱调用将被阻􏱋，直到一个连接􏰕􏰵建立
  
BufferedReader in = new BufferedReader(
    new InputStreamReader(clientSocket.getInputStream()));
PrintWriter out = new PrintWriter(clientSocket.getOutputStream(), true); ← -- ❷ 这􏰍􏳱对􏱝流对象都派􏱔生于􏰩该套接字的􏳱流对象􏱝 
  
String request, response;
while ((request = in.readLine()) != null) { ← -- ❸ 􏰻处理循环开始
	if ("Done".equals(request)) {
	break; ← -- 如􏰀客􏰡􏰢发送􏰃“Done”，则退出􏰻理􏱼环
	}
	response = processRequest(request); ← -- ❹ 请􏰿求被传􏱄递给服务器􏰠的处􏰻理方法􏳛

	out.println(response); ← --服务器的响应被发送给了客户端                                          }
```

这段代码将同时只能处理一个连接，如果要**处理多个链接，就需要开启多个线程**

![image-20210703110204228](https://tva1.sinaimg.cn/large/008i3skNly1gs3kwpqqt7j31420kw0wi.jpg)

问题：

1. 任何时候都有大量的线程处于休眠状态，资源浪费
2. 需要为每个线程的调用栈分配内存
3. 及时JVM物理上可以支持非常大数量的线程，但是当线程达到一定数量级 10 000，所带来的上下文开销依然很麻烦



幸运的是： 还有一种方案

#### 2. Java NIO

NIO的核心`选择器`

![image-20210703110654642](https://tva1.sinaimg.cn/large/008i3skNly1gs3l1r1szvj312t0u00xg.jpg)

优点：

1. 使用较少的线程便可以处理许多链接，因此也减少了内存管理和上下文切换所带来的开销
2. 没有I/O操作的时候，线程也能被用来干其他事

#### 3. Netty

下面的关键特性，有些是技术性的，有些是关于架构或设计哲学的

![image-20210703111315030](https://tva1.sinaimg.cn/large/008i3skNly1gs3l8cgbg5j30zl0u0n74.jpg)

Netty核心组件

- Channel
- 回调
- Future
- 事件和ChannelHandler

##### 3.1 Channel：它代表一个到实体的开􏰐放连接，如读操作和写操作

JavaNIO的一个基本构造，实体可以是<u>一个􏲛硬件设备、一个文件、一个网络套接字􏰌或者一个或多个不同的I/O操做􏱏的程序组件</u>， **可以把`Channel`看做是一个传输数据的载体，所以它是可以被打开和关闭的**

##### 3.2 回调：就是回调方法

##### 3.3 Future

JDK预置􏰃interface java.util.concurrent.Future ，但是其􏰉提供􏲯的实现，只允􏳅􏰇手动检查􏴳􏰎对应的操􏱏是否已􏰂完成，􏰌或者一直阻􏱋直到它完成。这是非常􏴴繁琐的，所􏰉以Netty提􏲯􏰃它自己的 实现——ChannelFuture ，用于在执行异步操作的时候调用

这里解决的方法是提供了一个`ChannelFutureListener` 注册到事件上，如果完成了会通知

##### 3.4 事件和ChannelHandler

Netty是一个网络编程框架􏰦􏰗􏰛􏰜，􏰉所以**事件是按照它们􏰈与入站和出站数据􏳱的相关性进行分类**􏱍的。这些事件包括：

- 连接已被激活或者连接失活
- 数据读取
- 用户事件
- 错误事件

`channelHandler`

![image-20210703114612178](https://tva1.sinaimg.cn/large/008i3skNly1gs3m6mz03kj315q0ckmzq.jpg)

Netty的ChannelHandler 为处理器提供了基本的抽象，如图1-3所示。目前􏰁你可以认为每􏰶个Channel-Handler 的实的实例都类似于一种为了响应特定事件而被执行的`回调`

### 第二章：第一款Netty应用程序

本章：

- 设置开发环境
- 编写Echo服务器和客户端
- 构建并测试应用程序

引入依赖

```java
				<dependency>
            <groupId>io.netty</groupId>
            <artifactId>netty-all</artifactId>
            <version>4.1.19.Final</version>
        </dependency>
```

### 1. 编写Echo服务器

所有的Netty服务器都需要下面两个部分

- 至少一个 ChannelHandler
- 引导——这是配置服务器的启动代码

服务器端代码

```java
import io.netty.bootstrap.ServerBootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.util.CharsetUtil;

import java.net.InetSocketAddress;

public class EchoServer {
    private final int port;

    public EchoServer(int port) {
        this.port = port;
    }

    public static void main(String[] args)
            throws Exception {
        // 服务器监听端口号
        int port = 8080;
        new EchoServer(port).start();
    }

  	//引导
    public void start() throws Exception {
        // NioEventLoopGroup是处理I/O操作的多线程事件循环
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            // ServerBootstrap是一个用于设置服务器的引导类。
            ServerBootstrap b = new ServerBootstrap();
            b.group(group)
                    .channel(NioServerSocketChannel.class) // 使用NioServerSocketChannel类，用于实例化新的通道以接受传入连接
                    .localAddress(new InetSocketAddress(port)) // 设置服务器监听端口号
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new EchoServerHandler()); // 添加请求处理
                        }
                    });
            // 绑定到端口和启动服务器
            ChannelFuture f = b.bind().sync();
            System.out.println(EchoServer.class.getName() +
                    " started and listening for connections on " + f.channel().localAddress());
            f.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully().sync();
        }
    }
}


//实现一个Handler
@ChannelHandler.Sharable
class EchoServerHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        //super.channelRead(ctx, msg);
        System.out.println("server: channel read");
        ByteBuf buf = (ByteBuf)msg;

        System.out.println(buf.toString(CharsetUtil.UTF_8));

        ctx.writeAndFlush(msg);

        ctx.close();

        //buf.release();
    }


    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        //super.exceptionCaught(ctx, cause);
        cause.printStackTrace();
        ctx.close();
    }
}

```

回顾一下

- EchoServerHandler 实现􏰃业务逻辑

- main()方法引导了服务器

  􏱒导过􏰗中􏰉需要的步􏵏如下:

  - 创􏰕一个ServerBootstrap 的实例􏱇以引导和绑定服务器􏰠
  - 创􏰕并分配一个NioEventLoopGroup 实例进行事件的􏰻管理，如接受新连接以及读/写数据
  - 指定􏰟服务器绑􏰠􏵒定的本地􏰚的InetSocketAddress
  - 􏰞用一个EchoServerHandler 的实􏱇初始化􏰶每一个新的Channel
  - 用ServerBootstrap.bind() 方法以绑定服务器

􏰞用一个EchoServerHandler 的实􏱇初始化􏰶一个新的Channel ; 􏱱用ServerBootstrap.bind() 方􏳛以􏵒定􏰟务􏰠。

### 2 编写Echo客户端

```java
import io.netty.bootstrap.Bootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.util.ReferenceCountUtil;

public class EchoClient {
    public static void main(String[] args) {
        new EchoClient().clientStart();    //启动客户端
    }

    private void clientStart() {
        EventLoopGroup workers = new NioEventLoopGroup();
        Bootstrap b = new Bootstrap();
        b.group(workers)
                .channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<SocketChannel>() {

                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        System.out.println("channel initialized!");
                        ch.pipeline().addLast(new ClientHandler());
                    }
                });

        try {
            System.out.println("start to connect...");
            ChannelFuture f = b.connect("127.0.0.1", 8080).sync();

            f.channel().closeFuture().sync();

        } catch (InterruptedException e) {
            e.printStackTrace();

        } finally {
            workers.shutdownGracefully();
        }

    }
}

class ClientHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        System.out.println("channel is activated.");

        final ChannelFuture f = ctx.writeAndFlush(Unpooled.copiedBuffer("HelloNetty".getBytes()));
        f.addListener(new ChannelFutureListener() {
//            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                System.out.println("msg send!");
                //ctx.close();
            }
        });


    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        try {
            ByteBuf buf = (ByteBuf)msg;
            System.out.println(buf.toString());
        } finally {
            ReferenceCountUtil.release(msg);
        }
    }
}
```

两个类代码类似，有些许差异，看清楚怎么实现通信就好。 回顾一下本节的要点：

- 为初始化客户端，创􏰕􏰃一个Bootstrap 实例; 
- 为进行事件􏰻处理分配了􏰃一个NioEventLoopGroup 实例，其中事件􏰻处理包括创建新动力连接，以及处理出站和入站的数据
- 为􏰟服务连接创􏰕建一个InetSocketAddress实例􏱇; 
- 当连接被􏰕􏰵时，一个EchoClientHandler 实􏱇会被安装到(该Channel 的) ChannelPipeline 中;
- 􏰋一切都设置完成后，􏱱用Bootstrap.connect() 方􏳛连接到远􏰗节点;

---

### 第三章：Netty的组件和设计

主要介绍了核心组件，以及各组件间的关系

### 3.1 `Channel`,`EventLoop`和`ChannelFuture`

- Channel—— Socket
- EventLoop—— 控制流、多线程处理、并发
- ChannelFuture—— 异步通知