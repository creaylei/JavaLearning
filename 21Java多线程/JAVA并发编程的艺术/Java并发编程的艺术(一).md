# Java并发编程的艺术(一)

[toc]

# 并发编程的挑战

## 1. 上下文切换

### 1. 什么是上线文切换

单核处理器也支持多线程执行代码，CPU通过给每个线程分配CPU时间片来实现这个机制。 

**时间片**一般是几十毫秒(ms)，所以感觉不到切换。

**上下文切换**：cpu执行A任务后，会切换到B任务，但是会记下A任务的状态，如果B任务执行完了，会再加载A任务的状态，（有点类似入栈出栈），从保存到再加载就是一次上下文切换。

### 2. 多线程不一定快

累加1W次，单线程要明显快。 这是因为多线程有上线文切换。

当100W次后，多线程快了起来

### 3. 如何减少上下文切换

- 无锁并发编程：多线程竞争锁时，会引起上线文切换，可以使用一些方法避免使用锁。
- CAS算法：Java的 Atomic包，用cas来更新数据，不用加锁
- 使用最少线程：如无必要，不使用多线程
- 使用协程：单线程里实现多任务的调度，并在单线程里维持多任务的切换

## 2. 死锁

死锁引起系统不可用

避免死锁

- 避免一个线程获取多个锁
- 一个锁内，尽量持有一个资源
- 尝试使用定时锁  `lock.tryLock(timeout)`
- 对于数据库锁，加锁和解锁必须在一个链接里，否则会出现解锁失败的情况

## 3. 资源限制

**举例：**带宽为2M/s,  每个线程速度1M/s, 并发开10个线程，不会达到10M/s

对于资源限制的一些方案：

- 硬件：使用集群，既然单机跑不动，那就集群起来
- 软件：使用资源池将连接复用，数据库连接池，线程池等，还有socket的多路复用

# 2 Java并发机制的底层原理

Volatile 和 sychronized： volatile更轻量，优先使用

### volatile

为了确保共享变量的可见性和一致性；

为了提高系统的处理速度，处理器不直接和内存通信。而是先将数据加载到自己的L1、L2缓存中，进行处理，但是操作完后不知道何时写入。  这时候如果是`volatile` 修饰的会立刻写入，其他线程会用嗅探机制判断当前引用是否变化，然后将缓存置为无效

### synchronized

java中每一个对象都可作为锁。具体表现为3种形式

1. 对于普通**同步方法**，锁当前实例对象
2. 对于**静态同步方法**，锁当前类的Class对象
3. 对于同步方法块，锁当前`synchronized` 括号里配置的对象

这是jvm层面实现的锁，JVM基于进入和退出Monitor对象来实现方法同步和代码块的同步。

- 代码块的同步是 `monitorenter` 和 `monitorexit` 实现的，代码编译后，会在开始出插入 `monitorenter`, 会在结束处和异常处插入 `monitorexit`。 
- 方法的同步没有详细说，但是也可以用这种方法实现

这里需要认识下Java对象头

![image-20200608140818304](https://tva1.sinaimg.cn/large/007S8ZIlly1gfkumcerl4j31oh0u0dwk.jpg)

如果是对象的话，前端的数据统称为 Mark Word，具体如下

![image-20200607155231852](https://tva1.sinaimg.cn/large/007S8ZIlly1gfjs0i30bfj31ie0u0tqy.jpg)

### 有趣的各种锁：真的有趣

线程获取锁和释放锁是很耗费资源的。为了减少性能消耗，引入“偏向锁”和“轻量级锁”，jdk 1.6中锁一共有4种状态，从低到高依次为：

- 无锁
- 偏向锁
- 轻量级锁
- 重量级锁

这几个状态会随着竞争情况逐渐升级，锁可以升级但是不能被降级

#### 2.1 偏向锁

HotSpot作者研究发现，大多数情况下，锁不仅不存在多线程竞争，而且总由一个线程反复获取。

> 这句话反复读几遍，有没有什么想法？给当前对象锁标记为这个线程，下次获得的时候，不用再通过CAS等方式去抢占，就省了好多资源，事实上也是这样做的

当一个线程访问同步块并获取锁的时候，会在我们上面所说的对象头和栈帧中的锁记录中存储锁偏向的线程id （锁偏向，也就是偏心，偏心哪个 哈哈）。

以后该线程在进入和退出同步块的时候，只需要简单的测试一下对象头的Mark word里面是否存储 指向当前线程id的偏向锁。 

- 如果测试成功，则表示已经获取了锁。
- 如果测试失败，再测试一个偏向锁的标志位是否设置为1，即是否是一个偏向锁，如果不是。 则使用CAS竞争锁；如果是，则尝试使用CAS将对象头的偏向锁指向当前线程。

来一个流程图看看吧

![image-20200608144153237](https://tva1.sinaimg.cn/large/007S8ZIlly1gfkvl8rtdkj30ns0qmwgm.jpg)

很清晰是吧，我也觉得，哈哈

**偏向锁的撤销：** 偏向锁使用了一种等到竞争出现才会撤销的方式。否则一直是前一个线程获得。

#### 2.2 轻量级锁

结合上面的图来看

![image-20200608150951449](https://tva1.sinaimg.cn/large/007S8ZIlly1gfkwecqsqfj31sa0hqdpl.jpg)

轻量级锁的加锁和解锁

加锁：

1. 线程在执行同步块之前，会先在当前线程的栈帧中创建用于存储锁记录的空间。
2. 将对象头的Mark Word复制到锁记录中，称为 displace Mark Word
3. 尝试使用CAS将对象头中的Mark word替换为指向锁记录的指针

解锁：反过来

1. 使用CAS将锁记录中的Mark Word替换到对象头
2. 如果成功，表示没有竞争；如果失败，表示存在竞争，锁就会膨胀为**重量级锁。** 

#### 2.3 重量级锁

一旦锁升级为重量级锁，就不会再恢复到轻量级状态，这个状态下，其他线程尝试获取这个锁都会被阻塞。当持有锁的线程释放后，唤醒所有竞争线程，新一轮争夺

### 锁的优缺点

![image-20200608152041860](https://tva1.sinaimg.cn/large/007S8ZIlly1gfkwpmp4d5j31s80ji1ax.jpg)

## 原子操作实现原理

原子操作：不可分割的最小操作，来看几个术语

![image-20200608152820848](https://tva1.sinaimg.cn/large/007S8ZIlly1gfkwxlk2gcj31s80r8kis.jpg)

处理器保证从内存中读或写一个字节是原子的，但是复杂操作处理器不能把保证。这里引入新的知识点：

- 总线锁定: 一个lock标识。 当其他线程看到lock标识，会被阻塞住。 解决i++问题

  总线锁将cpu和内存之间的通信锁住了，其他处理器不能操作内存中的数据。所以开销大, 索引引入缓存锁定。

- 缓存锁定: 频繁使用的内存会缓存在CPU的 L1、L2、L3高速缓存中，那么原子操作就可以在高速缓存中进行（我去，还可以这样，厉害了）。然后写回去的时候，由于缓存一致性，可以保证原子操作

  两种场景不能用：

  1. 当操作的数据不能被缓存在处理器内部，或者跨多个缓存行时
  2. 有些处理器不支持

## JAVA实现原子操作

**用CAS和锁实现**

常用的 AtomicLong， AtomicBoolean(原子方式更新boolean值)

2 CAS的问题：

1. ABA问题， A被改为B，又被改回来了。  线程在修改的时候不知道，这种要使用版本号机制来完善
2. 循环时间长开销大
3. 只能保证一个共享变量的原子操作。 多个变量取巧的办法是将两个变量和为一个变量

3 使用锁

java里面除了偏向锁，都使用了CAS，即进入锁的时候采用CAS方式，释放锁的时候还使用CAS

# JAVA内存模型

## 3.1 java内存模型的基础(JMM java memory model)

并发编程模型的两个关键问题：

1. 线程之间如何通信：
2. 线程之间如何同步

通信方式有两种：

- 共享内存: 线程之间共享程序的公共状态，通过写-读公共内存隐式通信，JAVA采用这种(想想堆)
- 消息传递: 线程通过发送消息来显示通信

同步：不同线程的操作，维持相对顺序

- 共享内存的情况下，是显式发生的。由程序员来显式指定方法的顺序
- 消息传递并发模型中，消息发送必须在消息接收之前。是隐式进行的

JMM定义了线程和主内存的抽象关系：线程间共享变量存储在主内存，每个线程都有一个私有的本地内存（实际不存在，是个抽象概念）。

## 3.2 重排序

编译器和处理器为了优化程序性能而对指令性能进行重排序。  提高处理速度

## 3.3 顺序一致性

## 3.4 Volatile内存语义

1. 线程A写一个Volatile变量，实质上是A向接下来将要读这个Volatile变量的某个线程发出了(其对共享变量做出修改)的消息
2. 线程B读一个Volatile变量，实质上的B接收了之前某个线程发出的消息
3. A写一个Volatile变量，B读这个volatile变量，这个过程实质上是A通过主内存向线程B发送消息。

> 通过内存屏障来实现

## 3.5 锁的内存语义

1. 线程A释放锁，实质上是线程A向接下来要获取这个锁的线程发送了(对共享变量所做修改)消息
2. 线程B获取锁，实质上是B接收到了之前某个线程发出的消息
3. A释放，B获取，这个过程实质上是A通过主内存向B发送消息

这里类似上面的Volatile语义

线程释放锁的时候，JMM会把线程对应的本地内存中的共享变量刷到主内存中。

## 举例

这里来看下`ReentrantLock` 

ReentrantLock的使用

 

```
public class ReentrantLockExample {
    int a = 0 ;
    ReentrantLock lock = new ReentrantLock();
    public void writer() {
        lock.lock();          //加锁
        try {
            a++;
        } finally {
            lock.unlock();    //释放锁
        }
    }
}
```

lock加锁, unlock释放锁。 ReentrantLock的实现依赖JAVA同步器框架`AbstractQueuedSynchronizer` （AQS）。 使用了一个 volatile state来维护同步状态。

![image-20200616115834940](https://tva1.sinaimg.cn/large/007S8ZIlly1gftzttrhssj30j60cewew.jpg)

ReentrantLock里面有两个内部类，公平锁和非公平锁，默认采用非公平锁。除非在构造方法传true，才会使用公平锁

公平锁：调用轨迹 如下

加锁1. Lock.lock()    2. FairSync.lock() 3. AQS.acquire(int arg) 4. ReentrantLock.tryAcquire()

解锁:1. Lock.unLock()   2. AQS.release(int arg) 3. ReentrantLock.tryRelease()

### 公平锁和非公平锁内存语义——小节

JDK文档说 CAS同时具有volatile读和volatile写的语义。

- 公平和非公平锁释放锁的时候，都要写一个volatile变量state
- 公平锁获取时，首先读volatile变量
- 非公平锁获取时，首先用CAS更新volatile值，这个操作同时具有volatile读和volatile写内存语义

### 3.6 Concurrent包

Concurrent包的源代码实现，有一个通用的实现模式

1. 声明volatile变量
2. 使用CAS原子条件更新来实现线程之间的同步
3. 同时配合以volatile读/写 和 CAS所具有volatile读和写的内存语义来实现线程间的通信

AQS, 非阻塞数据结构 以及 原子变量类 都是使用上面的模式来实现的。 而ConCurrent高层类又是使用这3个基础类实现的

![image-20200616142305363](https://tva1.sinaimg.cn/large/007S8ZIlly1gfu407lqw2j30y00o40x3.jpg)

## 3.6 final域内存语义

## 3.7 happens-before

## 3.8 双重检查锁定和延迟初始化：重要

1. 懒加载，不安全的

 

```
public class Singleton1 {
    public static Singleton1 singleton1;
    public static Singleton1 getInstance() {       //提供全局访问
        if(singleton1 == null) {                   //懒加载 A执行
            return new Singleton1();               // B线程执行
        }
        return singleton1;
    }
}
```

这里存在线程安全问题，如果A执行到第4行，B执行到第5行。线程A可能会看到singleton1引用的对象还没完成【初始化】，会有问题， 那么优化

1. 对`getInstance()`方法同步处理

 

```
public class Singleton2 {
    public static Singleton2 singleton2;
    private Singleton2() {}
    public static synchronized Singleton2 getSingleton2() {
        if(singleton2 == null) {
            return new Singleton2();
        }
        return singleton2;
    }
}
```

`synchronized`同步会导致性能开销，如果`getInstance()` 是多个线程频繁调用，那么开销就比较大，如果调用频率比较低，那么就完全胜任。 那么继续优化

1. 早起`synchronized`会有巨大开销，所以有了双重检查

 

```
public class Singleton6 {
    public static Singleton6 singleton6;
    private Singleton6() {}
    public static Singleton6 getSingleton6() {
        if(singleton6 == null) {    //第一次检查
            synchronized (Singleton6.class) {   //加锁
                if(singleton6 == null) {     //第二次检查
                    return new Singleton6();  //问题的根源
                }
            }
        }
        return singleton6;
    }}
```

这里看起来很美好，但是到第7行， singleton6 不为null时候，其引用的对象可能还没有完成初始化

这里的原因是： new Singleton6() 在jvm中会转化为下面3行代码

 

```
memory = allocate();   //1分配空间
ctorInstance(memory);  //2初始化对象
instance = memory;     //3.设置instance指向刚才分配的地址
```

问题产生的原因是2和3会重排序。 如下

 

```
memory = allocate();   //1分配空间
instance = memory;     //2.设置instance指向刚才分配的地址
                                                 这里注意，此时对象还没有被初始化
ctorInstance(memory);  //3 初始化对象
```

知道了是重排序导致的问题后，想到两种解决办法：

1. 阻止重排序 (volatile)
2. 可以重排序，但是不能让其他线程“看到”这个重排序 (内部类holder)

第一种方式是将instance实例声明为volatile格式的：该做法本质是禁止指令重排序

第二种做法的基于：JVM在类的初始化阶段(在Class被加载后，且被线程使用之前)，会执行类的初始化。在执行初始化期间，JVM会去获取一个锁。这个锁**可以同步多个线程对同一个类的初始化。**

 

```
public class Singleton4 {
        //将构造内部类话，
    private static class SingletonHolder{
        private static Singleton4 singleton4 = new Singleton4();
    }
    private Singleton4() {}
    public static Singleton4 getSingleton4() {
        return SingletonHolder.singleton4;
    }
}
```

上面这个方法是实质是，A和B线程都调用构造方法，但是只有A能进行构造，并且A内也会重排序，但是B都看不到。所以是安全的。

一个补充

> **1、对于静态方法，由于此时对象还未生成，所以只能采用类锁；**
>
> **2、只要采用类锁，就会拦截所有线程，只能让一个线程访问。**
>
> **3、\**\**对于对象锁（this），如果是同一个实例，就会按顺序访问，但是如果是不同实例，就可以同时访问。**
>
> **4、如果对象锁跟访问的对象没有关系，那么就会都同时访问。**

后面看JAVA并发编程基础