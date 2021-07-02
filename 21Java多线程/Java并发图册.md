# Java并发图册

[toc]

> 日拱一兵著， 同名公众号「日拱一兵」

并发编程3个核心问题：**分工、同步/写作、互斥**， 分工是设计，同步和互斥是实现

## 1. **并发问题三大根源**：可见性、原子性、有序性

作为"资本家"，你要尽可能的榨取 CPU，内存与 IO 的剩余价值，但三者完成任务的速度相差很大，CPU > 内存 > IO，CPU 是天，那内存就是地，内存是天，那 IO 就是地，那怎样平衡三者，提升整体速度呢?

1. **<font color="orange">CPU 增加缓存，还不止一层缓存，平衡内存的慢</font>**
2. <font color="orange">**CPU 能者多劳，通过分时复用，平衡 IO 的速度差异** </font>
3. <font color="orange">**优化编译指令**</font>

上面的方式貌似解决了木桶短板问题，但同时这种解决方案也伴随着产生新的<font color="orange">**可⻅性，原子性，和有序性**</font>的问题



| 可见性     | **一个线程对共享变量的修改，另外一个线程能够立刻看到，我们称为可⻅性** |
| ---------- | ------------------------------------------------------------ |
| **原子性** | **所谓原子操作是指不会被线程调度机制打断的操作;这种操作一旦开始，就一 直运行到结束，中间不会有任何 context switch** |
| 有序性     |                                                              |

### 可见性：主内存和私有内存

### 原子性: 大象放进冰箱

### 有序性: CPU的重排序

如果让你写单例模式，你怎么写？双重检查锁实现单例的例子

```java
public class Singleton {
 private static Singleton uniqueSingleton;

 private Singleton() {
 }

 public Singleton getInstance() {
     if (null == uniqueSingleton) {
         synchronized (Singleton.class) {
             if (null == uniqueSingleton) {
                 uniqueSingleton = new Singleton();   // error,会被JVM重排序
             }
         }
     }
     return uniqueSingleton;
 }
}
```

实现单例模式的时候，双重检查锁还不太够，还需要加上 `volatile` 或者 `final` ，  `volatile` 可以理解是防止重排序， 

小问题：final是怎么解决的呢?

---

## 2.三大问题的解法

- 为了解决 CPU，内存，IO 的短板，增加了缓存，但这导致了可⻅性问题 
- 编译器/处理器 `擅自`优化 ( Java代码在编译后会变成 Java 字节码, 字节码被 类加载器加载到 JVM 里, JVM 执行字节码, 最终需要转化为汇编指令在 CPU 上执行) ，导致有序性问题

矛盾点：

1. 作为程序员不想写出`bug` 影响KPI，所以希望内存模型易于理解、易于编程， 需要 <font color= "Tangerine">强内存模型</font>
2. 作为`编译器`和`处理器`，不想别人说自己慢，所以需要内存模型的束缚越少越好，可以由他们`擅自优化`，就需要一个<font color= "Tangerine">弱内存模型</font>

### 2.1 **有序性可⻅性，Happens-before 来搞定**

六大原则：

1. 程序顺序性原则：**同一个线程中**的每个操作，happens-before 于该线程中的任意后续操作

   > 同一个线程中，操作是有序的。 隐含语义，只要执行结果不被改变，怎么排序都行

2. Volatile变量原则: 对一个`volatile` 域的写, happens-before 于任意后续对这个 `volatile` 域的读

   | **能否重排序** | 第二个操作 | 第二个操作  | 第二个操作  |
   | -------------- | ---------- | ----------- | ----------- |
   | **第一个操作** | 普通读/写  | volatile 读 | volatile 写 |
   | 普通读/写      | -          | -           | NO          |
   | volatile 读    | NO         | NO          | NO          |
   | volatile 写    | -          | NO          | NO          |

   **两个关键点：**

   1. 如果第二个操作为 volatile 写，不管第一个操作是什么，都不能重排序，**这就 确保了 volatile 写之前的操作不会被重排序到 volatile 写之后** 

   2. 如果第一个操作为 volatile 读，不管第二个操作是什么，都不能重排序，**这确 保了 volatile 读之后的操作不会被重排序到 volatile 读之前**

      > volatile 内存语义的实现是应用到了 「内存屏障」, 内存屏障后面再说

3. 传递性原则：如果 A happens-before B, 且 B happens-before C, 那么 A happens-before C 

4. **监视器锁规则：对一个锁的解锁 happens-before 于随后对这个锁的加锁**

   > 这个说起来很简单，但是做的时候总会有问题，多画图，明确两点： 1. 锁的是什么 2. 屏障区是什么

5. `start()` 原则:如果线程 A 执行操作 `ThreadB.start() `(启动线程B), 那么 A 线程的 `ThreadB.start() `操作 happens-before 于线程 B 中的任意操作，也就是说，主 线程 A 启动子线程 B 后，子线程 B 能看到主线程在启动子线程 B 前的操作， 看个程序就秒懂了

6. `join()` 原则：如果线程 A 执行操作 `ThreadB.join() `并成功返回, 那么线程 B 中的任意操作 happens-before 于线程 A 从 `ThreadB.join()` 操作成功返回，**和 start 规则刚 好相反**，主线程 A 等待子线程 B 完成，当子线程 B 完成后，主线程能够看到 子线程 B 的赋值操作

小节:

-  happens-before 解决的是有序性问题，是原则
- start 和 join 也是解决主线程和子线程通信的方式之一
- 从内存语义的⻆度来说, volatile 的 写-读 与锁的 释放-获取 有相同的内存效 果;volatile 写和锁的释放有相同的内存语义; volatile 读与锁的获取有相同的 内存语义， **(敲黑板了) volatile 解决的是可⻅性问题，synchronized 解决的是原子性问题，**这绝对不是一回事

### 2.2 如何解决原子性问题

原子性问题的源头是：`线程切换`， 新规矩： 同一时刻只有一个线程执行

补充一下 `synchronized` 三种用法

```java
public class ThreeSync {
	private static final Object object = new Object();
	public synchronized void normalSyncMethod(){ //临界区
	}
	public static synchronized void staticSyncMethod(){ //临界区
	}
	public void syncBlockMethod(){ synchronized (object){
		//临界区
	} }
}
```

三种 synchronized 锁的内容有一些差别:

- 对于普通同步方法，锁的是当前实例对象，通常指 this 
- 对于静态同步方法，锁的是当前类的 Class 对象，如 ThreeSync.class 
- 对于同步方法块，锁的是 synchronized 括号内的对象

**临界区：我们把需要互斥执行的代码看成为临界区，如何用锁保护有效的临界区才是关键**，这直接关系到你是否会写出并发的 bug，无论是隐式 锁/内置锁 (synchronized) 还是显示锁 (Lock) 的使用都是在找寻这种关系

![image-20210702165011968](https://tva1.sinaimg.cn/large/008i3skNly1gs2pco4lsvj30r00p4q69.jpg)

#### **重中之重**：锁什么和保护什么

1. 我们锁的是什么？
2. 我们保护的又是什么？

需要**勾画草图时，明确找到这个关系**

1. 编写串行程序时，是不建议 try...catch 整个方法的，这样如果出现问是很难定 位的，道理一样，我们要用锁精确的锁住我们要保护的资源就够了，其他无意 义的资源是不要锁的

2. 锁保护的东⻄越多，临界区就越大，一个线程从走入临界区到走出临界区的时 间就越⻓，这就让其他线程等待的时间越久，这样并发的效率就有所下降，其 实这是涉及到锁粒度的问题，后续也都会做相关说明

#### 小节

1. 解决原子性问题，就是要互斥，就是要保证中间状态对外不可⻅

2. 锁是解决原子性问题的关键，明确知道我们锁的是什么，要保护的资源是什么，更重要的要知道你的锁能否保护这个受保护的资源(图中的箭头指向)
3. **有效的临界区是一个入口和一个出口，多个临界区保护一个资源，也就是一个 资源有多个并行的入口和多个出口，这就没有起到互斥的保护作用，临界区形 同虚设**

4. 锁自己家⻔能保护资源就没必要锁整个小区，如果锁了整个小区，这严重影响 其他业主的活动(锁粒度的问题)

## 3. `volatile`详解

复习下： happens-before 的原则之一: volatile变量规则

> 对一个 volatile 域的写, happens-before 于任意后续对这个 volatile 域的读

**这个表格是否还记得？**

| **能否重排序** | 第二个操作 | 第二个操作  | 第二个操作  |
| -------------- | ---------- | ----------- | ----------- |
| **第一个操作** | 普通读/写  | volatile 读 | volatile 写 |
| 普通读/写      | -          | -           | NO          |
| volatile 读    | NO         | NO          | NO          |
| volatile 写    | -          | NO          | NO          |

这是 JMM 针对编译器定制的 volatile 重排序的规则，那 JMM 是怎样禁止 重排序的呢?答案是**内存屏障**

### 内存屏障(**Memory Barriers / Fences**)