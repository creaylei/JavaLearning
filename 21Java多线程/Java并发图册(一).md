# Java并发图册（1-5）

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

> 为了实现 volatile 的内存语义，编译器在生成字节码时，会在指令序列中插入 内存屏障来禁止特定类型的处理器重排序

这句话有点抽象，试着想象内存屏障是一面高墙，**如果两个变量之间有这个屏障， 那么他们就不能互换位置(重排序)了，变量有读(Load)有写(Store)**，操作有前有后， JMM 就将内存屏障插入策略分为 4 种:

1. 在每个 volatile 写操作的前面插入一个 StoreStore 屏障 
2. 在每个 volatile 写操作的后面插入一个 StoreLoad 屏障 
3. 在每个 volatile 读操作的后面插入一个 LoadLoad 屏障 
4. 在每个 volatile 读操作的后面插入一个 LoadStore 屏障

![image-20210705160808323](https://tva1.sinaimg.cn/large/008i3skNly1gs64zubzqlj31800lotgh.jpg)

### volatile读内存语义

这里要理解 主存 和 线程本地内存

**当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效，从主存中进行读取**

写完立刻刷主存，读的时候直接读主存，这就是 `volatile`

## 4. 一把锁锁多个资源

### 保护没有关系的多个资源

各自加锁即可，相互不干扰

### 保护存在关系的多个资源

画图， 锁是什么？ 保护什么?

单个资源和多个无关联资源的情形都很好处理，为各自资源创建相应的锁就好

如果多个资源有关联，为了让锁起到保护作用，我们需要将锁的粒度变大，比如将 this 锁变成了 Account.class 锁。

### 如何避免死锁

`知己知彼，百战百胜` ，死锁是怎么产生的？ `Coffman`总结了四个条件会发生死锁，

#### **Coffman条件**

1. **互斥条件：**一段时间一个资源只能一个进程使用。如果此时还有其他进程请求，那么只能等待，直到被释放
2. **请求和保持条件：**进程已经持有一个资源，但又需要新的资源，而该资源被其他进程占有，此时请求进程阻塞。但又对已获得的资源保持不放
3. **不可被剥夺条件**：指进程已获得的资源，在未使用完之前，不能被剥夺，只能在使用完时由自己释放。
4. **环路等待条件：**指在发生死锁时，必然存在一个进程——资源的环形链，即进程集 合{P1，P2，···，Pn}中的 P1 正在等待一个 P2 占用的资源;P2 正在等待 P3 占用 的资源，......，Pn 正在等待已被 P0 占用的资源。

**第一个条件是并发编程的根基，不能被改变，其他三个都能被改变**

#### 1. 破坏「请求和保持条件」

> 有这么一句话：任何软件工程遇到的问题都可以通过增加一个中间层来解决

在银行转账案例中，每个柜员都可以取放账本，很容易出现互相等待的情况。要想破坏请求和保持条件，就要一次性拿到所有资源。

#### 2. 破坏「不可剥夺条件」

`sychronized` 不可剥夺，那么`Java`提供的 `Lock` 就粉墨登场了。Java 显示锁支持通知 (notify/notifyall)和等待(wait)，也就是说该功能可以实现喊一嗓子 **老铁，铁蛋儿的 账本先给我用一下，用完还给你** 的功能，这个后续将到 Java SDK 相关内容时会做 说明

#### 3. 破坏「环路等待」

我们只需要将**资源序号大小排序**获取就会解决这个问题，将环路拆除

**附加说明：**

在实际业务中，关于 Account 都会是数据库对象，我们可以通过事务或数据库的乐 观锁来解决的。另外分布式系统中，账本管理员这个⻆色的处理也可能会用 redis 分 布式锁来解决.

在处理破坏请求和保持条件时，我们使用的是 while 循环方式来不断请求锁的时 候，在实际业务中，我们会有 timeout 的设置，防止无休止的浪费 CPU 使用率

另外大家可以尝试使用阿里开源工具 Arthas 来查看 CPU 使用率，线程等相关问 题，github 上有明确的说明

## 5. `volatile` 和 `synchronized`区分

#### 相同点

遇到线程不安全的问题，在解决「共享变量内存可见性」问题上

`synchronized`：

- 【进入】synchronized 块的内存语义是把在 synchronized 块内使用的变量从 线程的工作内存中清除，从主内存中读取
- 【退出】synchronized 块的内存语义事把在 synchronized 块内对共享变量的 修改刷新到主内存中

`volatile`:

- 线程在【读取】共享变量时，会先清空本地内存变量值，再从主内存获取最新值
- 线程在【写入】共享变量时，不会把值缓存在寄存器或其他地方(就是刚刚说的所谓的「工作内存」)，而是会把值刷新回主内存

总结来说： **就是对于可见性，   取的时候从主内存取， 写的时候，直接写主内存**

![image-20210706115654272](https://tva1.sinaimg.cn/large/008i3skNly1gs73cpi17rj311o0f84eg.jpg)

#### 不同点

在解决多线程「原子性」问题上， `synchronized`能解决原子性问题， `volatile` 不能解决

如： i++ , 

- `synchronized` 会依次执行完，其他线程才允许访问
- `volatile` 不排他，不能保证”原子性“

#### 用法

如果写入变量值不依赖变量当前值，那么就可以用 volatile

synchronized 是排他的，线程排队就要有切换，这个切换就好比上面的例子，要完 成切换，还得记准线程上一次的操作，很累CPU大脑，这就是通常说的上下文切换 会带来很大开销

volatile 就不一样了，它是非阻塞的方式，所以在解决**共享变量可⻅性问题**的时候， **volatile 就是 synchronized 的弱同步体现了**

## 6 换个角度理解线程生命周期

> 作者博客地址 https://dayarch.top/p/spring-bean-lifecycle-creation.html

线程生命周期容易和操作系统生命周期混淆，做下区分

![image-20210706170555835](https://tva1.sinaimg.cn/large/008i3skNly1gs7ca8zltlj31960r24qp.jpg)

#### **操作系统通用线程状态：**

1. 初始状态：线程已被创建，但是还不被允许分配CPU执行。**注意，这个被创建其实是属于编程 语言层面的，实际在操作系统里，真正的线程还没被创建**， 比如 Java 语言中的 new Thread()。
2. 可运行状态：线程可以分配CPU执行，**这时，操作系统中线程已经被创建成功了**
3. 运行状态：操作系统会为 处在可运行状态的线程 分配CPU时间片，被 CPU 临幸后，处在可运行 状态的线程就会变为运行状态
4. 休眠状态：如果处在运行状态的线程调用 某个阻塞的API 或 等待某个事件条件可用 ，那么线程就 会转换到休眠状态，**注意:此时线程会释放CPU使用权，休眠的线程永远没有机会 获得CPU使用权，只有当等待事件出现后，线程会从休眠状态转换到可运行状态**
5. 终止状态：线程执行完 或者 出现异常 (被interrupt那种不算的哈，后续会说)就会进入终止状 态，正式走到生命的尽头，没有起死回生的机会

#### Java语言线程状态

Thread类源码中，定义了一个枚举类 State

```java
1. NEW
2. RUNNABLE
3. BLOCKED
4. WAITING
5. TIMED_WAITING 
6. TERMINATED
```

JAVA语言中

- 将通用线程状态的 可运行状态 和 运行状态 合并为 Runnable 
- 将休眠状态 细分为三种 ( **BLOCKED / WAITING / TIMED_WAITING** ); 反过来理 解这句话，就是这三种状态在操作系统的眼中都是休眠状态，同样**不会获得 CPU使用权**

除去线程的生死，我们只 **要玩转 RUNNABLE 和 休眠状态 的转换**就可以了，编写并发程序也多数是这两种状 态的转换

### 强烈推荐 Arthas

> https://arthas.aliyun.com/doc/install-detail.html#id1

