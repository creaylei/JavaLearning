# JAVA并发图册(3-3)

[toc]

## 11. 图解AQS(独占式) 和 ReentrantLock

> 共享式类似

看看能不能回答下面的问题

![image-20210818114716281](https://tva1.sinaimg.cn/large/008i3skNly1gtkspxidh2j61260pujuk02.jpg)

### 1. 为什么要设计Lock

如果只有`synchronized()`多好, 1. 锁当前实例 2. 锁类 3. 锁静态方法

```java
public class ThreeSync {

 private static final Object object = new Object();

 public synchronized void normalSyncMethod(){
  //临界区
 }

 public static synchronized void staticSyncMethod(){
  //临界区
 }

 public void syncBlockMethod(){
  synchronized (object){
   //临界区
  }
 }
}
```

有一个轮子还要再造一个轮子，为什么？

 Coffman 总结的四个可以发生死锁的情形 ，其中【不可剥夺条件】是指

> 线程已经获得资源，在未使用完之前，不能被剥夺，只能在使用完时自己释放

要想破坏这个条件，**就需要具有申请不到进一步资源就释放已有资源的能力**, 显然synchronized 不具备这种功能，申请不到资源就会阻塞，我们没法改变。   这就需要Lock了



### 2. Lock

提供了3个方法响应中断

![image-20210818115507300](https://tva1.sinaimg.cn/large/008i3skNly1gtksy3lntgj61e00oedjr02.jpg)

使用范式

```java
Lock lock = new ReentrantLock();
lock.lock();
try{
 ...
}finally{
 lock.unlock();
}
```

这里有两个标准

1. **标准1—finally 中释放锁** 
2. **标准2—在 try{} 外面获取锁** 

> 想想为什么在外面获取锁？ 主要有两方面：
>
> 1. 没有获取到锁，就抛异常，最后释放锁会有问题 。  未曾拥有何谈释放
> 2. 如果try里面获取锁，发生异常，那么没获取到，仍然会finally 释放锁，此时别的线程获取到锁，影响了。(无故释放)

#### Lock怎么锁的呢？

- `lock.lock()` 获取锁，“等同于” synchronized 的 moniterenter指令
- `lock.unlock()` 释放锁，“等同于” synchronized 的 moniterexit 指令

那么lock怎么做到的呢?

其实很简单，比如在 ReentrantLock 内部维护了一个 volatile 修饰的变量 state，通过 CAS 来进行读写（最底层还是交给硬件来保证原子性和可见性），如果CAS更改成功，即获取到锁，线程进入到 try 代码块继续执行；如果没有更改成功，线程会被【挂起】，不会向下执行

但 Lock 是一个接口，里面根本没有 state 这个变量的存在：

![image-20210818142454001](https://tva1.sinaimg.cn/large/008i3skNly1gtkx9zne5ej615i0j80ua02.jpg)

它怎么处理这个 state 呢？很显然需要一点设计的加成了，接口定义行为，具体都是需要实现类的

> Lock 接口的实现类基本都是通过【聚合】了一个【队列同步器】的子类完成线程访问控制的

### 3 AQS : `AbstractQueuedSynchronizer`

为什么要用AQS呢? 怎么进一步理解锁和同步器的关系呢？

![image-20210818142905268](https://tva1.sinaimg.cn/large/008i3skNly1gtkxeaph6aj613w0dgq5402.jpg)

从 AQS 的类名称和修饰上来看，这是一个抽象类，所以从设计模式的角度来看同步器一定是基于【模版模式】来设计的，使用者需要继承同步器，实现自定义同步器，并重写指定方法，随后将同步器组合在自定义的同步组件中，并调用同步器的模版方法，而这些模版方法又回调用使用者重写的方法

理解上面这句话，我们只需要知道下面两个问题就好了

1. 哪些是自定义同步器可重写的方法？
2. 哪些是抽象同步器提供的模版方法？

#### **同步器可重写的方法**

同步器提供的可重写方法只有5个，这大大方便了锁的使用者：

![image-20210818143206905](https://tva1.sinaimg.cn/large/008i3skNly1gtkxhg4n2gj61560d4aco02.jpg)

按理说，需要重写的方法也应该有 abstract 来修饰的，为什么这里没有？原因其实很简单，上面的方法我已经用颜色区分成了两类：

- `独占式`
- `共享式`

自定义的同步组件或者锁不可能既是独占式又是共享式，为了避免强制重写不相干方法，所以就没有 abstract 来修饰了

表格方法描述中所说的`同步状态`就是上文提到的有 volatile 修饰的 state，所以我们在`重写`上面几个方法时，还要通过同步器提供的下面三个方法（AQS 提供的）来获取或修改同步状态：

![image-20210818145942533](https://tva1.sinaimg.cn/large/008i3skNly1gtkya5ow1dj614208y75g02.jpg)

独占式和共享式区别就是： state 0—> 1   和 state 0—> N的区别

所以你看到的 `ReentrantLock` `ReentrantReadWriteLock` `Semaphore(信号量)` `CountDownLatch` 这几个类其实仅仅是在实现以上几个方法上略有差别，其他的实现都是通过同步器的模版方法来实现的

#### **同步器提供的模版方法**

上面我们将同步器的实现方法分为独占式和共享式两类，模版方法其实除了提供以上两类模版方法之外，只是多了`响应中断`和`超时限制` 的模版方法供 Lock 使用，来看一下

![image-20210818150453046](https://tva1.sinaimg.cn/large/008i3skNly1gtkyfjnbhrj617e0g2gpb02.jpg)

注意到了吗，这里提供的模板方法都是final修饰，表示不能被重写。 实现自定义同步器方法的时候就用到这些方法

**稍微归纳下**

![image-20210818173309523](https://tva1.sinaimg.cn/large/008i3skNly1gtl2ptuicnj61ho0r8n0p02.jpg)

看一段代码

```java
package top.dayarch.myjuc;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.AbstractQueuedSynchronizer;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;

/**
 * 自定义互斥锁
 *
 * @author tanrgyb
 * @date 2020/5/23 9:33 PM
 */
public class MyMutex implements Lock {

 // 静态内部类-自定义同步器
 private static class MySync extends AbstractQueuedSynchronizer{
  @Override
  protected boolean tryAcquire(int arg) {
   // 调用AQS提供的方法，通过CAS保证原子性
   if (compareAndSetState(0, arg)){
    // 我们实现的是互斥锁，所以标记获取到同步状态（更新state成功）的线程，
    // 主要为了判断是否可重入（一会儿会说明）
    setExclusiveOwnerThread(Thread.currentThread());
    //获取同步状态成功，返回 true
    return true;
   }
   // 获取同步状态失败，返回 false
   return false;
  }

  @Override
  protected boolean tryRelease(int arg) {
   // 未拥有锁却让释放，会抛出IMSE
   if (getState() == 0){
    throw new IllegalMonitorStateException();
   }
   // 可以释放，清空排它线程标记
   setExclusiveOwnerThread(null);
   // 设置同步状态为0，表示释放锁
   setState(0);
   return true;
  }

  // 是否独占式持有
  @Override
  protected boolean isHeldExclusively() {
   return getState() == 1;
  }

  // 后续会用到，主要用于等待/通知机制，每个condition都有一个与之对应的条件等待队列，在锁模型中说明过
  Condition newCondition() {
   return new ConditionObject();
  }
 }

  // 聚合自定义同步器
 private final MySync sync = new MySync();


 @Override
 public void lock() {
  // 阻塞式的获取锁，调用同步器模版方法独占式，获取同步状态
  sync.acquire(1);
 }

 @Override
 public void lockInterruptibly() throws InterruptedException {
  // 调用同步器模版方法可中断式获取同步状态
  sync.acquireInterruptibly(1);
 }

 @Override
 public boolean tryLock() {
  // 调用自己重写的方法，非阻塞式的获取同步状态
  return sync.tryAcquire(1);
 }

 @Override
 public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
  // 调用同步器模版方法，可响应中断和超时时间限制
  return sync.tryAcquireNanos(1, unit.toNanos(time));
 }

 @Override
 public void unlock() {
  // 释放锁
  sync.release(1);
 }

 @Override
 public Condition newCondition() {
  // 使用自定义的条件
  return sync.newCondition();
 }
}
```

如果打开IDE, 你会发现上文提到的 `ReentrantLock` `ReentrantReadWriteLock` `Semaphore(信号量)` `CountDownLatch` 都是按照这个结构实现，所以我们就来看一看 AQS 的模版方法到底是怎么实现锁

### 4 AQS实现分析

从上面的代码中，你应该理解了`lock.tryLock()` 非阻塞式获取锁就是调用自定义同步器重写的 `tryAcquire()` 方法，通过 CAS 设置state 状态，不管成功与否都会马上返回；那么 lock.lock() 这种阻塞式的锁是如何实现的呢？

**有阻塞就有排队、有排队就有队列**

AQS中的队列，是CLH变体的**虚拟双向队列**（FIFO）——概念了解就好，不要记

队列中每个排队的个体就是一个 Node，所以我们来看一下 Node 的结构









## 8. //待写

获取锁的过程分三部分：

1. 如果acquire成功，那直接返回
2. 否则进入 双向队列
3. 进入队列判断是否挂起

![AQS获取锁的过程](https://tva1.sinaimg.cn/large/008i3skNly1gt74wl7z0vj31io0u0acu.jpg)



## 9. 



## 看到P276：Future_Task