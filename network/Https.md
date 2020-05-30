# Https流程梳理

https://zhuanlan.zhihu.com/p/27395037

https = http + SSL/TLS

HyperText transfer Protocol   +  Secure Socket Layer     /  Transport Layer Security，传输层安全

## 涉及知识

- 对称加密

- 非对称加密

- Hash算法

- 数字签名

  签名就是在信息的后面再加上一段内容（信息经过hash后的值），可以证明信息没有被修改过。hash值一般都会加密后（也就是签名）再和信息一起发送，以保证这个hash值不被修改。



## Http访问过程

### What？

http主要是应用层传输协议，以特定的格式进行消息排列，然后传输。接收方接收到后，以已知的协议格式拆解。

![image-20200530083132403](https://tva1.sinaimg.cn/large/007S8ZIlly1gfa6b4fro6j30uc0badj5.jpg)

### Why？

传输协议

### How？

发起一个http请求，会先编排成http消息格式， 然后加上 tcp/ip 头。  进行三次握手，然后Ip头。

接收方，拆ip头， 拆 tcp头， 解析http内容。

注意这里的顺序。



## Why Https

http文本都是明文传输，很容易被中间人截获。 伪造，攻击。

- 伪造身份
- 伪造信息

在Tcp上层添加 SSL/TLS层

### How

![image-20200530084035066](https://tva1.sinaimg.cn/large/007S8ZIlly1gfa6kj0wqcj30z20lyqa2.jpg)

用通常的A和B来讲就是

1. A发送请求，给出协议版本号，  一个随机数 Cr（client Random）， 以及客户端支持的加密算法
2. B 确认使用的加密算法， 给出数字证书，以及自己生成的随机数 Sr （Server Random）
3. A确认 B的证书是否有效， 然后生成一个  Premaster Random （即后面的私钥），并使用数字证书中的公钥，加密 Pr，发给B
4. B使用自己的私钥，解密 A发来的信息， 获取 Pr
5. A和B通过 Cr  Sr Pr 和约定的加密算法，来生成 对话秘钥，即对称秘钥，后面就用这个来通信

### 核心点

1. 整个过程需要三个随机数，  密码的安全取决于黑客能不能破解用公钥加密的第三个随机数。

2. 证书的获取一般来自CA，可信的第三方
3. 默认公钥算法是RSA