# JAVA并发图册(2-3)

[toc]

## 7. 面试官问创建多少个线程合适？

试着回答一下下面脑图里面的问题

![image-20210707195434816](https://tva1.sinaimg.cn/large/008i3skNly1gs8ms28qh2j310c0jsgw7.jpg)

### 为什么要用多线程？用多少个合适呢？

使用多线程就是在**正确的场景**下通过设置**正确个数的线程** 来最大化程序的运行速度。

这就是充分利用 CPU 和 I/O的利用率，充分榨取。  这就是从 【定性】和【定量】两个角度分析

#### 【定性】正确的场景

- CPU密集型 ： 单核不适合多线程， 如果是「多核」CPU，完全可以最大化利用CPU核心数
- IO密集型：线程等待比率越高， 需要越多

**总结：线程等待时间所占比例越高，需要越多线程；线程CPU时间所占比例越高，需要越少线程**

#### 【定量】多少个呢

- CPU密集型：   CPU核数 + 1 ，   1是backup
- I/O密集型： 【单核】最佳线程数 = (1/CPU利用率) = 1 + （IO耗时 / CPU耗时） 
                       【多核】最佳线程数 = CPU核心数 * (1/CPU利用率) = CPU核心数 * (1+(IO耗时 / CPU耗时))

初始按这样就行，不过后面还要调优

上面有两个疑问点：

1. 怎么知道具体 IO 耗时 和 CPU耗时呢？ 
2. 怎么查看CPU利用率？ 

 APM工具 ： 1. Skywalking 2. CAT 3. zipkin

> 灵魂拷问
>
> 1. 我们已经知道多少个线程合适了，为什么还要搞一个线程池？
> 2. 创建一个线程都需要做哪些事？为什么说开销很大？

## 8. 手动创建线程很简单，为什么要使用线程池？

一个很朴素的观点：如果用了另一个，那么其一定有比当前方式好的地方，即优点。 换句话说，当前方式存在某些缺点

### 手动创建线程的缺点

1. 不受控风险 ： 每个人针对不同的业务都可以手动创建线程，并且创建的标准还不一样。 系统运行起来，所有线程都疯狂抢占资源，无组织、无纪律。

   如果有位神奇的小伙伴，每个请求都创建一个线程，当大量请求铺天盖地而来，**就好比一个正规木马程序，内存被无情榨干**

2. 频繁创建开销大

   有什么开销？这里去分析啊new一个对象和new 一个线程。

   ```java
   new一个对象有3步
   1. 分配一块内存M
   2. 在内存M上初始会一个对象
   3. 将M的地址赋值给引用对象 Obj
   
   这也是 new 一个对象不是原子操作，容易导致并发问题的原因
   ```

   new一个线程需要调用操作系统的内核API。 看看创建并启动一个线程，JVM在背后做了什么

   ![image-20210813165117676](https://tva1.sinaimg.cn/large/008i3skNly1gtf9erd9naj61460d2mzt02.jpg)

   创建一个线程（即使什么都不做），也需要 `1M`的空间

手动创建线程的弊端我们是知道了，那么该怎么办呢？   答案当然就是线程池了

### 什么是线程池

池化的思想：数据库连接池、实例池。 简而言之**就是为了最大化收益，最小化风险，将资源统一在一起管理的思想**。

关键点： `ThreadPoolExecutor` 方法

```java
/**
     * Creates a new {@code ThreadPoolExecutor} with the given initial
     * parameters.
     *
     * @param corePoolSize the number of threads to keep in the pool, even
     *        if they are idle, unless {@code allowCoreThreadTimeOut} is set
     * @param maximumPoolSize the maximum number of threads to allow in the
     *        pool
     * @param keepAliveTime when the number of threads is greater than
     *        the core, this is the maximum time that excess idle threads
     *        will wait for new tasks before terminating.
     * @param unit the time unit for the {@code keepAliveTime} argument
     * @param workQueue the queue to use for holding tasks before they are
     *        executed.  This queue will hold only the {@code Runnable}
     *        tasks submitted by the {@code execute} method.
     * @param threadFactory the factory to use when the executor
     *        creates a new thread
     * @param handler the handler to use when execution is blocked
     *        because the thread bounds and queue capacities are reached
     * @throws IllegalArgumentException if one of the following holds:<br>
     *         {@code corePoolSize < 0}<br>
     *         {@code keepAliveTime < 0}<br>
     *         {@code maximumPoolSize <= 0}<br>
     *         {@code maximumPoolSize < corePoolSize}
     * @throws NullPointerException if {@code workQueue}
     *         or {@code threadFactory} or {@code handler} is null
     */
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

上面英文解释如果还不清楚的话，看看下面流程图

![image-20210813165903084](https://tva1.sinaimg.cn/large/008i3skNly1gtf9msopx2j60z80r875r02.jpg)



### 不可忽略的拒绝策略

不能忽略拒绝策略， 我们很难估计未来准确并发量，所以必须选择合适的拒绝策略。 默认提供了4种

![image-20210813170417300](https://tva1.sinaimg.cn/large/008i3skNly1gtf9s8ijurj618e07qaaz02.jpg)

见名思义：

1. AbortPolicy： 抛弃策略，默认的拒绝策略，会抛出`RejectedExecutionException` 拒绝
2. CallerRunsPolicy:  调用提交的线程自己执行该任务
3. DiscardOldestPolicy： 抛弃最早的任务，从队列头移除老任务，加入新任务
4. DiscardPolicy： 相当大胆的策略，直接抛弃，没有任何异常抛出

---

### 禁用`Executors`创建线程池

很多人都听过这个问题吧，大名鼎鼎的阿里巴巴JAVA开发手册禁止使用`Executors`创建线程池

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
    
    
    
    /**
     * Creates a {@code LinkedBlockingQueue} with a capacity of
     * {@link Integer#MAX_VALUE}.
     */
    public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE);
    }
```

传入默认的一个`LinkedBlockingQueue<Runnable>()` 是一个无界队列，这么大的等待队列也非常耗内存。  并且默认的拒绝策略也是直接丢弃。

总之，`Executors`太过于理想化，还是自己老老实实创建

### 灵魂拷问

1. 线程池的缺点是什么？
2. 为什么不建议所有的业务共用一个线程池?有什么缺点？
3. 给线程池设定指定前缀，有哪些方式？

## 9. 等待/通知机制，和想象中的完全不一样

内容概览：

1. 并发编程为什么有等待/通知机制
2. 如何应用  等待 / 通知 机制
3. 为什么说尽量用 notifyAll
4. 什么时候用 notify不会有问题
5. MESA 监视器模型的简单介绍

### 9.1 为什么要有`等待/通知机制`

上回提到，解决死锁的思路之一就是：破坏`请求和保持条件`

没有`等待/通知机制` 之前，是通过死循环的方式，不断请求， 耗费CPU

> 转账场景中，拿到  this， target 两个账本，拿不到就一直循环试

### 9.2 如何应用

理想情况下转账场景

- 如果A拿不到所需的所有账本，就不再问 （自己阻塞自己）
- 如果B归还了A所需的账本，就通知A  (通知等待线程  notifyAll / notify)

JAVA语言中，内置的 `sychronized` 和 `wait() , notify(), notifyAll()` ,就能实现等待 / 通知机制

一个重要的图

![image-20210817154544476](https://tva1.sinaimg.cn/large/008i3skNly1gtjtzrazsjj60yk0qkjte02.jpg)

几个面试知识点：

1. 一个锁对应一个【入口等待队列】，不同锁的入口等待队列没任何关系。  举个栗子：到医院耳鼻喉科和 眼科 看大夫没有任何冲突
2. `wait() , notify(), notifyAll()`  要在 `sychronized` 内部使用。 并且，如果锁的对象是this，那么就要用 `this.wait() , this.notify(), this.notifyAll()`  ， 否则JVM会抛出 `java.lang.IllegalMonitorStateException()`。

有了铺垫之后，将死循环改为  等待 / 通知 机制有几个问题：

| 1 锁的是什么？           | 以转账举例： 锁 单例 AccountManager对象 |
| ------------------------ | --------------------------------------- |
| 2 线程要求的条件是什么？ | 转出、转入账户都可用                    |
| 3 何时等待？             | 条件不满足                              |
| 4 何时通知？             | 条件同时满足                            |

#### 引出一个使用`wait()`的标准范式

```java
sychronized() {
	while(条件不满足) {
		wait();
	}
}
```

这里两个点：

1. Wait()阻塞，  唤醒后从wait()继续往后执行，   这个就是就是这里不用if 而用 while 的原因，如果if唤醒，和获取条件有时间差，会导致不满足条件；而while会再次判断一下

2. 唤醒的时候尽量使用 `notifyAll()` 

   notifyAll() 不会遗漏等待队列中的线程，但是notify会随机唤醒一个，其他的有可能死掉

那什么时候用notify() 呢？

1. 所有线程拥有相同的等待条件
2. 所有线程被唤醒后，执行相同的操作
3. 只需要唤醒一个线程

notify()的一个典型应用是线程池， 对照下上面三个条件，理解了吗？

这里可以对比下 JUC中的 await() / signal()  —— wait() / notify()

### 9.3 MESA监视器模型

MESA监视器模型，每一个条件变量都对应一个条件等待队列。

> 其实就是上面那张图：贴一篇链接，https://blog.csdn.net/it_lihongmin/article/details/109190265



## 10. 贯穿并发编程的终端机制

![image-20210817194815095](https://tva1.sinaimg.cn/large/008i3skNly1gtk102jyvdj60wm0lawgk02.jpg)

### 1. JAVA中，【中断】是一种【协同】机制

就是中断是一种通知，至于是否中断由线程自己决定

### 2. 为什么有中断

多线程场景中，有的线程可能迷失（自旋浪费资源），这时候可以用其他线程在**适当**的时机，给一个中断通知。 被“中断”的线程可以选择在“适当”的时机，跳出，最大化利用资源

### 3. `interrupt()`  VS `isInterrupt()` VS `interrupted()`

java每个线程都有一个 boolean 类型的标识，代表是否有中断请求。  但是Thread类中没有，因为是native方法实现。

| 方法            | 功能                                                        | 场景                                                   |
| --------------- | ----------------------------------------------------------- | ------------------------------------------------------ |
| `interrupt()`   | 将上面的 boolean标识，设置为true                            |                                                        |
| `isInterrupt()` | 返回中断标识的结果，即 true / false                         |                                                        |
| `interrupted()` | 返回当前中断标识，然后清空中断标识。第二次调用一定返回false | 当一个线程可能被大量中断，并且你想确保只处理一次中断时 |

### 4 中断机制使用场景

- 关闭某个应用(关闭idea，不保存数据好吗？)
- 某个操作超过一定时间
- 多个线程做同一件事情，只要一个线程成功，其他都可以取消时
- 一组线程中的一个或多个出现错误导致整组无法继续时

### 5 注意事项

1. 如果遇到的是可中断的阻塞方法, 并抛出 InterruptedException，可以继续向方法调用栈的上层抛出该异常；如果检测到中断，则可清除中断状态并抛出 InterruptedException，使当前方法也成为一个可中断的方法
2. 若有时候不太方便在方法上抛出 InterruptedException，比如要实现的某个接口中的方法签名上没有 throws InterruptedException，这时就可以捕获可中断方法的 InterruptedException 并通过 Thread.currentThread.interrupt() 来重新设置中断状态。

这两个原则很好理解。总的来说，我们应该留意 InterruptedException，当我们捕获到该异常时，绝不可以默默的吞掉它，什么也不做，因为这会导致上层调用栈什么信息也获取不到。其实在编写程序时，捕获的任何受检异常我们都不应该吞掉



做了这么多铺垫，终于到关键部分啦！
