# JAVA并发图册(3-5)

[toc]

原文链接： [万字超强图文讲解AQS](https://zhuanlan.zhihu.com/p/323903756)

[一个经典的简单多线程题](https://leetcode-cn.com/problems/print-in-order/solution/si-chong-bu-tong-de-shi-xian-fang-shi-zong-jie-by-/)

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

#### 前置知识：Node和队列

从上面的代码中，你应该理解了`lock.tryLock()` 非阻塞式获取锁就是调用自定义同步器重写的 `tryAcquire()` 方法，通过 CAS 设置state 状态，不管成功与否都会马上返回；那么 lock.lock() 这种阻塞式的锁是如何实现的呢？

**有阻塞就有排队、有排队就有队列**

AQS中的队列，是CLH变体的**虚拟双向队列**（FIFO）——概念了解就好，不要记

AQS中定义了一个内部类Node, 队列中每个排队的个体就是一个 Node，所以我们来看一下 Node 的结构

![image-20210819104815710](https://tva1.sinaimg.cn/large/008i3skNly1gtlwmv8fghj61900p6jur02.jpg)

Node 有 prev  有  next， 还有个waitstatus，  可以想象一个双向队列了

![image-20210819105018659](https://tva1.sinaimg.cn/large/008i3skNly1gtlwoz1ph3j619k0f4jt002.jpg)



#### 独占式获取同步状态

> 这里先介绍下独占式获取同步状态，和释放同步状态，   共享式类型，知识 0-1，和 0-N的区别

故事从lock.lock开始

这段开始要看一些必要的代码，不多

```java
public void lock() {
 // 阻塞式的获取锁，调用同步器模版方法，获取同步状态
 sync.acquire(1);
}
```

进入**AQS的模版方法** acquire()

```java
public final void acquire(int arg) {
  // 调用自定义同步器重写的 tryAcquire 方法
 if (!tryAcquire(arg) &&
  acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
  selfInterrupt();
}
```

首先，也会尝试非阻塞的获取同步状态，如果获取失败（tryAcquire返回false），则会调用 `addWaiter` 方法构造 Node 节点（Node.EXCLUSIVE 独占式）并安全的（CAS）加入到同步队列【尾部】

```java
//加到队列里
private Node addWaiter(Node mode) {
       // 构造Node节点，包含当前线程信息以及节点模式【独占/共享】
        Node node = new Node(Thread.currentThread(), mode);
       // 新建变量 pred 将指针指向tail指向的节点
        Node pred = tail;
       // 如果尾节点不为空
        if (pred != null) {
           // 新加入的节点前驱节点指向尾节点
            node.prev = pred;

           // 因为如果多个线程同时获取同步状态失败都会执行这段代码
            // 所以，通过 CAS 方式确保安全的设置当前节点为最新的尾节点
            if (compareAndSetTail(pred, node)) {
               // 曾经的尾节点的后继节点指向当前节点
                pred.next = node;
               // 返回新构建的节点
                return node;
            }
        }
       // 尾节点为空，说明当前节点是第一个被加入到同步队列中的节点
       // 需要一个入队操作
        enq(node);
        return node;
    }

    private Node enq(final Node node) {
       // 通过“死循环”确保节点被正确添加，最终将其设置为尾节点之后才会返回，这里使用 CAS 的理由和上面一样
        for (;;) {
            Node t = tail;
           // 第一次循环，如果尾节点为 null
            if (t == null) { // Must initialize
               // 构建一个哨兵节点，并将头部指针指向它
                if (compareAndSetHead(new Node()))
                   // 尾部指针同样指向哨兵节点
                    tail = head;
            } else {
               // 第二次循环，将新节点的前驱节点指向t
                node.prev = t;
               // 将新节点加入到队列尾节点
                if (compareAndSetTail(t, node)) {
                   // 前驱节点的后继节点指向当前新节点，完成双向队列
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

你可能比较迷惑 enq() 的处理方式，进入该方法就是一个“死循环”，我们就用图来描述它是怎样跳出循环的

![image-20210819105759804](https://tva1.sinaimg.cn/large/008i3skNly1gtlwwz0vtyj619s0qk76v02.jpg)

有些同学可能会有疑问，为什么会有哨兵节点？

> 哨兵，顾名思义，是用来解决国家之间边界问题的，不直接参与生产活动。同样，计算机科学中提到的哨兵，也用来解决边界问题，如果没有边界，指定环节，按照同样算法可能会在边界处发生异常，比如要继续向下分析的 `acquireQueued()` 方法

```java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
           // "死循环"，尝试获取锁，或者挂起
            for (;;) {
               // 获取当前节点的前驱节点
                final Node p = node.predecessor();
               // 只有当前节点的前驱节点是头节点，才会尝试获取锁
               // 看到这你应该理解添加哨兵节点的含义了吧
                if (p == head && tryAcquire(arg)) {
                   // 获取同步状态成功，将自己设置为头
                    setHead(node);
                   // 将哨兵节点的后继节点置为空，方便GC
                    p.next = null; // help GC
                    failed = false;
                   // 返回中断标识
                    return interrupted;
                }
               // 当前节点的前驱节点不是头节点
               //【或者】当前节点的前驱节点是头节点但获取同步状态失败
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

获取同步状态成功会返回可以理解了，但是如果失败就会一直陷入到“死循环”中浪费资源吗？很显然不是，`shouldParkAfterFailedAcquire(p, node)` 和 `parkAndCheckInterrupt()` 就会将线程获取同步状态失败的线程挂起，我们继续向下看

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
       // 获取前驱节点的状态
        int ws = pred.waitStatus;
       // 如果是 SIGNAL 状态，即等待被占用的资源释放，直接返回 true
       // 准备继续调用 parkAndCheckInterrupt 方法
        if (ws == Node.SIGNAL)
            return true;
       // ws 大于0说明是CANCELLED状态，
        if (ws > 0) {
            // 循环判断前驱节点的前驱节点是否也为CANCELLED状态，忽略该状态的节点，重新连接队列
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
           // 将当前节点的前驱节点设置为设置为 SIGNAL 状态，用于后续唤醒操作
           // 程序第一次执行到这返回为false，还会进行外层第二次循环，最终从代码第7行返回
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

至此省略。。。。[万字超强图文讲解AQS](https://zhuanlan.zhihu.com/p/323903756)

来一张图概览一下

![image-20210819110610451](https://tva1.sinaimg.cn/large/008i3skNly1gtlx5h85yqj61ii0tydip02.jpg)

获取锁的过程就这样的结束了，先暂停几分钟整理一下自己的思路。我们上面还没有说明 SIGNAL 的作用， SIGNAL 状态信号到底是干什么用的？这就涉及到锁的释放了，我们来继续了解，整体思路和锁的获取是一样的， 但是释放过程就相对简单很多了

#### **独占式释放同步状态**

故事要从 unlock() 方法说起

```java
public void unlock() {
  // 释放锁
  sync.release(1);
 }
```

调用 AQS 模版方法 release，进入该方法

```text
public final boolean release(int arg) {
       // 调用自定义同步器重写的 tryRelease 方法尝试释放同步状态
        if (tryRelease(arg)) {
           // 释放成功，获取头节点
            Node h = head;
           // 存在头节点，并且waitStatus不是初始状态
           // 通过获取的过程我们已经分析了，在获取的过程中会将 waitStatus的值从初始状态更新成 SIGNAL 状态
            if (h != null && h.waitStatus != 0)
               // 解除线程挂起状态
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

查看 unparkSuccessor 方法，实际是要唤醒头节点的后继节点

```java
private void unparkSuccessor(Node node) {      
       // 获取头节点的waitStatus
        int ws = node.waitStatus;
        if (ws < 0)
           // 清空头节点的waitStatus值，即置为0
            compareAndSetWaitStatus(node, ws, 0);
      
       // 获取头节点的后继节点
        Node s = node.next;
       // 判断当前节点的后继节点是否是取消状态，如果是，需要移除，重新连接队列
        if (s == null || s.waitStatus > 0) {
            s = null;
           // 从尾节点向前查找，找到队列第一个waitStatus状态小于0的节点
            for (Node t = tail; t != null && t != node; t = t.prev)
               // 如果是独占式，这里小于0，其实就是 SIGNAL
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
           // 解除线程挂起状态
            LockSupport.unpark(s.thread);
    }
```

有同学可能有疑问：

> 为什么这个地方是从队列尾部向前查找不是 CANCELLED 的节点？ 先想一想，再看原文吧

到这里，关于独占式获取/释放锁的流程已经闭环了，但是关于 AQS 的另外两个模版方法还没有介绍

- `响应中断`
- `超时限制`

#### **独占式响应中断获取同步状态**

故事要从lock.lockInterruptibly() 方法说起

```java
public void lockInterruptibly() throws InterruptedException {
  // 调用同步器模版方法可中断式获取同步状态
  sync.acquireInterruptibly(1);
 }
```

#### **独占式超时限制获取同步状态**

这个很好理解，就是给定一个时限，在该时间段内获取到同步状态，就返回 true， 否则，返回 false。好比线程给自己定了一个闹钟，闹铃一响，线程就自己返回了，这就不会使自己是阻塞状态了

既然涉及到超时限制，其核心逻辑肯定是计算时间间隔，因为在超时时间内，肯定是多次尝试获取锁的，每次获取锁肯定有时间消耗，所以计算时间间隔的逻辑就像我们在程序打印程序耗时 log 那么简单

> nanosTimeout = deadline - System.nanoTime()

故事要从 `lock.tryLock(time, unit)` 方法说起

```java
public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
  // 调用同步器模版方法，可响应中断和超时时间限制
  return sync.tryAcquireNanos(1, unit.toNanos(time));
 }
```

#### Condition

![image-20210819111448016](https://tva1.sinaimg.cn/large/008i3skNly1gtlxeglfyuj60ym0u0tao02.jpg)

那故事就从 `lock.newnewCondition` 说起吧

```java
public Condition newCondition() {
  // 使用自定义的条件
  return sync.newCondition();
 }
```

自定义同步器重封装了该方法：

```java
Condition newCondition() {
   return new ConditionObject();
  }
```

ConditionObject 就是 Condition 的实现类，该类就定义在了 AQS 中，只有两个成员变量：

```java
/** First node of condition queue. */
private transient Node firstWaiter;
/** Last node of condition queue. */
private transient Node lastWaiter;
```

所以，我们只需要来看一下 ConditionObject 实现的 await / signal 方法来使用这两个成员变量就可以了

```java
public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
           // 同样构建 Node 节点，并加入到等待队列中
            Node node = addConditionWaiter();
           // 释放同步状态
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
               // 挂起当前线程
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```

这里注意用词，在介绍获取同步状态时，addWaiter 是加入到【同步队列】，就是上图说的入口等待队列，这里说的是【等待队列】，所以 addConditionWaiter 肯定是构建了一个自己的队列：

```java
private Node addConditionWaiter() {
            Node t = lastWaiter;
            
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
           // 新构建的节点的 waitStatus 是 CONDITION，注意不是 0 或 SIGNAL 了
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
           // 构建单向同步队列
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
```



在 Lock 中可以定义多个条件，每个条件都会对应一个 条件等待队列，所以将上图丰富说明一下就变成了这个样子：

![image-20210819111646413](https://tva1.sinaimg.cn/large/008i3skNly1gtlxgi7jfsj617i0u0wi202.jpg)

获取锁的过程分三部分：

1. 如果acquire成功，那直接返回
2. 否则进入 双向队列
3. 进入队列判断是否挂起

线程已经按相应的条件加入到了条件等待队列中，那如何再尝试获取锁呢？signal / signalAll 方法就已经排上用场了

```java
public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
```

Signal 方法通过调用 doSignal 方法，只唤醒条件等待队列中的第一个节点

```java
private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
               // 调用该方法，将条件等待队列的线程节点移动到同步队列中
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
```

继续看 `transferForSignal` 方法

```java
final boolean transferForSignal(Node node) {       
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        // 重新进行入队操作
        Node p = enq(node);
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
           // 唤醒同步队列中该线程
            LockSupport.unpark(node.thread);
        return true;
    }
```

所以我们再用图解一下唤醒的整个过程

![image-20210819111816579](https://tva1.sinaimg.cn/large/008i3skNly1gtlxi2awh2j619c0rstbg02.jpg)

### 5. 看看ReentrantLock是如何实现的

其他的锁实现，模式是一致的

独占式的典型应用就是 ReentrantLock 了，我们来看看它是如何重写这个方法的

![image-20210819112043092](https://tva1.sinaimg.cn/large/008i3skNly1gtlxkm8q6sj617g0pego002.jpg)

乍一看挺奇怪的，怎么里面自定义了三个同步器：其实 NonfairSync，FairSync 只是对 Sync 做了进一步划分：

![image-20210819112109007](https://tva1.sinaimg.cn/large/008i3skNly1gtlxl2450rj60we0i6my702.jpg)

#### **何为公平锁/非公平锁？**

![image-20210819112208231](https://tva1.sinaimg.cn/large/008i3skNly1gtlxm339n2j61ra0n044702.jpg)

其实没什么大不了，公平锁就是判断同步队列是否还有先驱节点的存在，只有没有先驱节点才能获取锁；而非公平锁是不管这个事的，能获取到同步状态就可以，就这么简单

公平锁保证了排队的公平性，非公平锁霸气的忽视这个规则，所以就有可能导致排队的长时间在排队，也没有机会获取到锁，这就是传说中的 **“饥饿”**

#### **如何选择公平锁/非公平锁？**

相信到这里，答案已经在你心中了，如果为了更高的吞吐量，很显然非公平锁是比较合适的，因为节省很多线程切换时间，吞吐量自然就上去了，否则那就用公平锁还大家一个公平

#### **可重入锁**

试想，如果是一个有 synchronized 修饰的递归调用方法，程序第二次进入被自己阻塞了岂不是很大的笑话，所以 synchronized 是支持锁的重入的

Lock 是新轮子，自然也要支持这个功能，其实现也很简单，请查看公平锁和非公平锁对比图，其中有一段代码：

```text
// 判断当前线程是否和已占用锁的线程是同一个
else if (current == getExclusiveOwnerThread())
```

仔细看代码， 你也许发现，我前面的一个说明是错误的，我要重新解释一下

![img](https://pic3.zhimg.com/v2-a50ee5b24e88398305114a8c6b2e6f2a_b.jpg)

重入的线程会一直将 state + 1， 释放锁会 state - 1直至等于0，上面这样写也是想帮助大家快速的区分

### **总结**

AQS 中我们介绍了独占式获取同步状态的多种情形：

- 独占式获取锁
- 可响应中断的独占式获取锁
- 有超时限制的独占式获取锁

本文是一个长文，说明了为什么要造 Lock 新轮子，如何标准的使用 Lock，AQS 是什么，是如何实现锁的，结合 ReentrantLock 反推 AQS 中的一些应用以及其独有的一些特性



## 12. 图解AQS(共享式)、以及 Semaphore

![image-20210821144527255](https://tva1.sinaimg.cn/large/008i3skNly1gtoeqabst3j616u0emdh702.jpg)

### 1. AQS共享式获取同步状态

想了解 AQS 的 xxxShared 的模版方法，只需要知道它是怎么控制 state 的就好了

### 2. AQS共享式获取同步状态源码分析

自定义同步器需要重写的方法

![image-20210821144816965](https://tva1.sinaimg.cn/large/008i3skNly1gtoet6ufwwj61700fe77b02.jpg)

AQS提供的模板方法

![image-20210821144848889](https://tva1.sinaimg.cn/large/008i3skNly1gtoetqv0l4j61780h2gpc02.jpg)

基本结构和独占式获取锁惊人的相似

故事从「模板方法」中的这里开始吧

```java
public final void acquireShared(int arg) {
          // 同样调用自定义同步器需要重写的方法，非阻塞式的尝试获取同步状态，如果结果小于零，则获取同步状态失败
        if (tryAcquireShared(arg) < 0)
              // 调用 AQS 提供的模版方法，进入等待队列
            doAcquireShared(arg);
    }
```

进入 `doAcquireShared` 方法：

```java
private void doAcquireShared(int arg) {
          // 创建共享节点「SHARED」，加到等待队列中
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
              // 进入“自旋”，这里并不是纯粹意义上的死循环，在独占式已经说明过
            for (;;) {
                  // 同样尝试获取当前节点的前驱节点
                final Node p = node.predecessor();
                  // 如果前驱节点为头节点，尝试再次获取同步状态
                if (p == head) {
                      // 在此以非阻塞式获取同步状态
                    int r = tryAcquireShared(arg);
                      // 如果返回结果大于等于零，才能跳出外层循环返回
                    if (r >= 0) {
                          // 这里是和独占式的区别
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

有没有发现这里类似独占式，都是进入队列，做个对比

![image-20210821145219000](https://tva1.sinaimg.cn/large/008i3skNly1gtoexe2i4aj61rb0u0jyh02.jpg)

 `setHeadAndPropagate(node, r)` 到底干了什么

![image-20210821145352228](https://tva1.sinaimg.cn/large/008i3skNly1gtoez0154kj61h80g476m02.jpg)

这里说的传播其实说的是 `propagate > 0` 的情况，道理也很简单，当前线程获取同步状态成功了，还有剩余的同步状态可用于其他线程获取，**那就要通知在等待队列的线程，让他们尝试获取剩余的同步状态**

我们分析了AQS 的模版方法，**还一直没说 `tryAcquireShared(arg)` 这个方法是如何被重写的**，想要了解这个，我们就来看一看共享式获取同步状态的经典应用 Semaphore

### 3. Semaphore

中文多翻译为信号量

其实就是信号标志（two flags），比如红绿灯，每个交通灯产生两种不同行为

Flag1-红灯：停车
Flag2-绿灯：行车
在 Semaphore 里面，什么时候是红灯，什么时候是绿灯，其实就是靠 tryAcquireShared(arg) 的结果来表示的

获取不到共享状态，即为红灯
获取到共享状态，即为绿灯

所以我们走近 Semaphore ，来看看它到底是怎么应用 AQS 的，又是怎样重写 `tryAcquireShared(arg)` 方法的

#### 先来看下类结构

![image-20210821145932158](https://tva1.sinaimg.cn/large/008i3skNly1gtof4wsoufj617q0u0tdw02.jpg)

相似的可怕，这里直接提速对比看公平和非公平两种重写的 `tryAcquireShared(arg)` 方法，没有意外，公平与否，就是判断是否有前驱节点

![image-20210821150117632](https://tva1.sinaimg.cn/large/008i3skNly1gtof6q4r3vj61j00bg41b02.jpg)

初始state是构造方法设置进去的

![image-20210821150237753](https://tva1.sinaimg.cn/large/008i3skNly1gtof84eewdj61hg0myn0802.jpg)

如果将`permits` 设置为1，即是`ReentrantLock`互斥锁了？没错，如下就是了

```java
static int count;
//初始化信号量
static final Semaphore s 
    = new Semaphore(1);
//用信号量保证互斥    
static void addOne() {
  s.acquire();
  try {
    count+=1;
  } finally {
    s.release();
  }
}
```

#### 小节： **Semaphore 可以允许多个线程访问一个临界区**，最终很好的做到一个**限流/限流/限流** 的作用，虽然 Semaphore 能很好的提供限流作用，说实话，Semaphore 的限流作用比较单一，我在实际工作中使用 Semaphore 并不是很多，如果真的要用高性能限流器，Guava RateLimiter 是一个非常不错的选择

跟上节奏，关于共享式获取同步状态，Semaphore 只不过是非常经典的应用，ReadWriteLock 和 CountDownLatch 日常应用还是非常广泛的，我们接下来就陆续聊聊它们吧

### 浅析ReentrantReadWriteLock

![image-20210821150828674](https://tva1.sinaimg.cn/large/008i3skNly1gtofe85vcwj61i00m6jtd02.jpg)

#### ReadWriteLock

直译过来是读写锁，之前提到的互斥锁都是排他锁，也就是说同一时刻只允许一个线程进行访问，当面对可共享读的业务场景，互斥锁显然是比较低效的一种处理方式。为了提高效率，读写锁模型就诞生了

关于读写锁模型就了下面这 3 条规定：

1. 允许多个线程同时读共享变量
2. 只允许一个线程写共享变量
3. 如果写线程正在执行写操作，此时则禁止其他读线程读共享变量



`ReadWriteLock` 是一个接口，其内部只有两个方法：

```java
public interface ReadWriteLock {
    // 返回用于读的锁
    Lock readLock();

    // 返回用于写的锁
    Lock writeLock();
}
```

### ReentrantReadWriteLock 类结构

![image-20210821151200057](https://tva1.sinaimg.cn/large/008i3skNgy1gtofhvvjicj61850u043s02.jpg)

基本示例

```java
public class ReentrantReadWriteLockCache {

	// 定义一个非线程安全的 HashMap 用于缓存对象
	static Map<String, Object> map = new HashMap<String, Object>();
	// 创建读写锁对象
	static ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
	// 构建读锁
	static Lock rl = readWriteLock.readLock();
	// 构建写锁
	static Lock wl = readWriteLock.writeLock();

	public static final Object get(String key) {
		rl.lock();
		try{
			return map.get(key);
		}finally {
			rl.unlock();
		}
	}

	public static final Object put(String key, Object value){
		wl.lock();
		try{
			return map.put(key, value);
		}finally {
			wl.unlock();
		}
	}
}
```

就这么简单。但是你知道的，AQS 的核心是锁的实现，即控制同步状态 state 的值，ReentrantReadWriteLock 也是应用AQS的 state 来控制同步状态的，那么问题来了：

> 一个 int 类型的 state 怎么既控制读的同步状态，又可以控制写的同步状态呢？

### 读写状态设计

如果要在一个 int 类型变量上维护多个状态，那肯定就需要拆分了。我们知道 int 类型数据占32位，所以我们就有机会按位切割使用state了。我们将其切割成两部分：

1. 高16位表示读
2. 低16位表示写

![image-20210821151415234](https://tva1.sinaimg.cn/large/008i3skNly1gtofk7v0r1j61io0qc41j02.jpg)

源码不做过多展示，稍微简介下：

写读是放在一个队列中，如果写等待在前面，那么读就必须等着



这里主要说明了 ReentrantReadWriteLock 是如何应用 state 做位拆分实现读/写两种同步状态的，另外也通过源码分析了读/写锁获取同步状态的过程，最后又了解了读写锁的升级/降级机制，相信到这里你对读写锁已经有了一定的理解。如果你对文中的哪些地方觉得理解有些困难，强烈建议你回看本文开头的两篇文章，那里铺垫了非常多的内容。接下来我们就看看在应用AQS的最后一个并发工具类 CountDownLatch 吧

### CountDownLatch 和 CyclicBarrier

![image-20210821151831930](https://tva1.sinaimg.cn/large/008i3skNgy1gtofoo39rsj612c0u0mzy02.jpg)

并发编程的三大核心是分工，同步和互斥。在日常开发中，经常会碰到需要在主线程中开启多个子线程去并行的执行任务，并且主线程需要等待所有子线程执行完毕再进行汇总的场景，这就涉及到分工与同步的内容了

在讲 有序性可见性，Happens-before来搞定 时，提到过 join() 规则，使用 join() 就可以简单的实现上述场景。 

```
thread1.start
thread2.start

thread1.join
thread2.join
```

我们来查看 join() 的实现源码：

![image-20210821152041451](https://tva1.sinaimg.cn/large/008i3skNly1gtofqwyadfj61f8094jsx02.jpg)

其实现原理是不停的检查 join 线程是否存活，如果 join 线程存活，则 wait(0) 永远的等下去，直至 join 线程终止后，线程的 this.notifyAll() 方法会被调用（该方法是在 JVM 中实现的，JDK 中并不会看到源码），退出循环恢复主线程执行。很显然这种循环检查的方式比较低效.

除此之外，使用 join() 缺少很多灵活性，比如实际项目中很少让自己单独创建线程（原因在 [我会手动创建线程，为什么要使用线程池?](https://dayarch.top/p/why-we-need-to-use-threadpool.html) 中说过）而是使用 Executor, 这进一步减少了 join() 的使用场景，所以 join() 的使用在多数是停留在 demo 演示上

> 那如何实现文中开头提到的场景呢？

#### CountDownLatch

翻译为 【向下的闸门】,看段代码

```java
@Slf4j
public class CountDownLatchExample {

    private static CountDownLatch countDownLatch = new CountDownLatch(2);

    public static void main(String[] args) throws InterruptedException {
        // 这里不推荐这样创建线程池，最好通过 ThreadPoolExecutor 手动创建线程池
        ExecutorService executorService = Executors.newFixedThreadPool(2);

        executorService.submit(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                log.info("Thread-1 执行完毕");
                //计数器减1
                countDownLatch.countDown();
            }
        });

        executorService.submit(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                log.info("Thread-2 执行完毕");
                //计数器减1
                countDownLatch.countDown();
            }
        });

        log.info("主线程等待子线程执行完毕");
        log.info("计数器值为：" + countDownLatch.getCount());
        countDownLatch.await();
        log.info("计数器值为：" + countDownLatch.getCount());
        log.info("主线程执行完毕");
        executorService.shutdown();
    }
}
```

结合上述示例的运行结果，相信你也能猜出 CountDownLatch 的实现原理了：

1. 初始化计数器数值，比如为2
2. 子线程执行完则调用 countDownLatch.countDown() 方法将计数器数值减1
3. 主线程调用 await() 方法阻塞自己，直至计数器数值为0（即子线程全部执行结束）

一个典型的场景是老师等所有学生都来了才开始上课。



CountDownLatch 暴露给使用者的只有 await() 和 countDown() 两个方法，前者是阻塞自己，因为只有获取同步状态才会才会出现阻塞的情况，那自然是在 await() 的方法内部会用到 tryAcquireShared()；有获取就要有释放，那后者 countDown() 方法内部自然是要用到 tryReleaseShared() 方法了。

小结
CountDownLatch 的实现原理就是这么简单，了解了整个实现过程后，你也许发现了使用 CountDownLatch 的一个问题：

计数器减 1 操作是一次性的，也就是说当计数器减到 0， 再有线程调用 await() 方法，该线程会直接通过，不会再起到等待其他线程执行结果起到同步的作用了

为了解决这个问题，贴心的 Doug Lea 大师早已给我们准备好相应策略 CyclicBarrier

#### CyclicBarrier

直译：环形屏障，来个例子

```java
@Slf4j
public class CyclicBarrierExample {

   // 创建 CyclicBarrier 实例，计数器的值设置为2
   private static CyclicBarrier cyclicBarrier = new CyclicBarrier(2);

   public static void main(String[] args) {
      ExecutorService executorService = Executors.newFixedThreadPool(2);
      int breakCount = 0;

         // 将线程提交到线程池
      executorService.submit(() -> {
         try {
            log.info(Thread.currentThread() + "第一回合");
            Thread.sleep(1000);
            cyclicBarrier.await();

            log.info(Thread.currentThread() + "第二回合");
            Thread.sleep(2000);
            cyclicBarrier.await();

            log.info(Thread.currentThread() + "第三回合");
         } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
         } 
      });

      executorService.submit(() -> {
         try {
            log.info(Thread.currentThread() + "第一回合");
            Thread.sleep(2000);
            cyclicBarrier.await();

            log.info(Thread.currentThread() + "第二回合");
            Thread.sleep(1000);
            cyclicBarrier.await();

            log.info(Thread.currentThread() + "第三回合");
         } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
         }
      });

      executorService.shutdown();
   }

}
```

结果：

![image-20210821153045083](https://tva1.sinaimg.cn/large/008i3skNly1gtog1dtrtnj61fi0bwn0u02.jpg)

线程1到达屏障，等待其他到达屏障，然后线程2到达屏障。 两个一起冲破栅栏，继续第二回合

![image-20210821153335296](https://tva1.sinaimg.cn/large/008i3skNly1gtog4c3huqj610e0oijsu02.jpg)



### 小总结：

`semaphore` ： 起限流作用，实际用处不大， 了解下 google的guava

`CountDownLatch` : 当计数器减到0时候一起执行， 但是重置比较麻烦，不常用

`CyclicBarrier` : 栅栏， 用的比较多。   加强版`CountDownLatch`, 可以重置初始值

本文讲解了 CountDownLatch 和 CyclicBarrier 的经典使用场景以及实现原理，以及在使用过程中可能会遇到的问题，比如将大的 list 拆分作业就可以用到前者，读取多个 Excel 的sheet 页，最后进行结果汇总就可以用到后者 （文中完整示例代码已上传）

最后，再形象化的比喻一下

CountDownLatch 主要用来解决一个线程等待多个线程的场景，可以类比旅游团团长要等待所有游客到齐才能去下一个景点
而 CyclicBarrier 是一组线程之间的相互等待，可以类比几个驴友之间的不离不弃，共同到达某个地方，再继续出发，这样反复

灵魂追问

1. 怎样拿到 CyclicBarrier 的汇总结果呢？
2. 线程池中的 Future 特性你有使用过吗？