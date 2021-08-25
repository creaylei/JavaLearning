# Java并发图册(4-4)

[toc]





## 13. Java Future

回答下面几个问题

![image-20210823143819008](https://tva1.sinaimg.cn/large/008i3skNly1gtqprh7ymyj610y0rcmzv02.jpg)

### Runnable 、 Callable对比

#### Runnable/Thread 创建线程的短板

- 没有参数
- 没有返回值
- 没办法抛出异常

没有返回值是最不能忍的，所以产生了Callable

#### Callable简介

```java
@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

#### 对比

![image-20210823145207542](https://tva1.sinaimg.cn/large/008i3skNly1gtqq5t9bxej61ii0ocgo502.jpg)

返回值和处理异常很好理解，另外，在实际工作中，我们通常要使用线程池来管理线程（*原因已经在 [为什么要使用线程池?](https://dayarch.top/p/why-we-need-to-use-threadpool.html) 中明确说明*），所以我们就来看看 ExecutorService 中是如何使用二者的

### ExecutorService

先来看一下 ExecutorService 类图

![image-20210823145413754](https://tva1.sinaimg.cn/large/008i3skNly1gtqq7zxj8zj61b10u0ae202.jpg)

我将上图标记的方法单独放在此处

```java
void execute(Runnable command);

<T> Future<T> submit(Callable<T> task);
<T> Future<T> submit(Runnable task, T result);
Future<?> submit(Runnable task);
```

可以看到，使用ExecutorService 的 execute() 方法依旧得不到返回值，而 submit() 方法清一色的返回 Future 类型的返回值

那么

- Future 到底是什么呢？
- 怎么通过它获取返回值呢？

### Future

![image-20210823145543489](https://tva1.sinaimg.cn/large/008i3skNly1gtqq9k0bjjj61ei0n8jtg02.jpg)

又是一个接口，从方法名上可以猜测分别为： 取消任务、获取结果、超时获取结果、判断是否取消、判断是否完成

来个例子看看

```java
@Slf4j
public class FutureAndCallableExample {

   public static void main(String[] args) throws InterruptedException, ExecutionException {
      ExecutorService executorService = Executors.newSingleThreadExecutor();

      // 使用 Callable ，可以获取返回值
      Callable<String> callable = () -> {
         log.info("进入 Callable 的 call 方法");
         // 模拟子线程任务，在此睡眠 2s，
         // 小细节：由于 call 方法会抛出 Exception，这里不用像使用 Runnable 的run 方法那样 try/catch 了
         Thread.sleep(5000);
         return "Hello from Callable";
      };

      log.info("提交 Callable 到线程池");
      Future<String> future = executorService.submit(callable);

      log.info("主线程继续执行");

      log.info("主线程等待获取 Future 结果");
      // Future.get() blocks until the result is available
      String result = future.get();
      log.info("主线程获取到 Future 结果: {}", result);

      executorService.shutdown();
   }
}
```

![image-20210823145911401](https://tva1.sinaimg.cn/large/008i3skNly1gtqqd6q3rvj61gc0bun0b02.jpg)

如果你运行上述示例代码，主线程调用 future.get() 方法会阻塞自己，直到子任务完成。我们也可以使用 Future 方法提供的 `isDone` 方法

相信到这里你已经对 `Future` 的几个方法有了基本的使用印象，但 `Future` 是接口，其实使用 `ExecutorService.submit()` 方法返回的一直都是 `Future` 的实现类 `FutureTask`

![image-20210823150510066](https://tva1.sinaimg.cn/large/008i3skNly1gtqqjdn0fmj61gq0ckdiw02.jpg)

### FutureTask

接下来我们就进入这个核心实现类一探究竟，类结构

![image-20210823150940429](https://tva1.sinaimg.cn/large/008i3skNly1gtqqo2m988j611c0r40uo02.jpg)

很神奇的一个接口，FutureTask 实现了 RunnableFuture 接口，而 RunnableFuture 接口又分别实现了 Runnable 和 Future 接口，所以可以推断出 FutureTask 具有这两种接口的特性：

有 Runnable 特性，所以可以用在 ExecutorService 中配合线程池使用，有 Future 特性，所以可以从中获取到执行结果

### FutureTask源码分析

如果你完整的看过 AQS 相关分析的文章，你也许会发现，阅读 Java 并发工具类源码，我们无非就是要关注以下这三点：

```
- 状态 （代码逻辑的主要控制）
- 队列 （等待排队队列）
- CAS （安全的set 值）
```

构造方法

```java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}
```

即便在 FutureTask 构造方法中传入的是 Runnable 形式的线程，该构造方法也会通过 Executors.callable 工厂方法将其转换为 Callable 类型：

```java
public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}
```

但是 FutureTask 实现的是 Runnable 接口，也就是只能重写 run() 方法，run() 方法又没有返回值，那问题来了：

> - FutureTask 是怎样在 run() 方法中获取返回值的？
> - 它将返回值放到哪里了？
> - get() 方法又是怎样拿到这个返回值的呢？

我们来看一下 run() 方法（关键代码都已标记注释）

```java
public void run() {
      // 如果状态不是 NEW，说明任务已经执行过或者已经被取消，直接返回
      // 如果状态是 NEW，则尝试把执行线程保存在 runnerOffset（runner字段），如果赋值失败，则直接返回
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
          // 获取构造函数传入的 Callable 值
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                  // 正常调用 Callable 的 call 方法就可以获取到返回值
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                  // 保存 call 方法抛出的异常
                setException(ex);
            }
            if (ran)
                  // 保存 call 方法的执行结果
                set(result);
        }
    } finally {        
        runner = null;       
        int s = state;
          // 如果任务被中断，则执行中断处理
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

`run()` 方法没有返回值，至于 `run()` 方法是如何将 `call()` 方法的返回结果和异常都保存起来的呢？其实非常简单, 就是通过 set(result) 保存正常程序运行结果，或通过 setException(ex) 保存程序异常信息

#### 核心是通过state来控制程序

FutureTask 对象被创建出来，state 的状态就是 NEW 状态，从上面的构造函数中你应该已经发现了，四个最终状态 NORMAL ，EXCEPTIONAL ， CANCELLED ， INTERRUPTED 也都很好理解，两个中间状态稍稍有点让人困惑:

COMPLETING: outcome 正在被set 值的时候
INTERRUPTING：通过 cancel(true) 方法正在中断线程的时候

![image-20210823151956971](https://tva1.sinaimg.cn/large/008i3skNly1gtqqyrxz9bj61d20qkq6402.jpg)

#### 烧水泡茶经典程序

![image-20210823152654217](https://tva1.sinaimg.cn/large/008i3skNly1gtqr5zt7nzj61ga0jsq4w02.jpg)

如上图：

- 洗水壶 1 分钟
- 烧开水 15 分钟
- 洗茶壶 1 分钟
- 洗茶杯 1 分钟
- 拿茶叶 2 分钟

最终泡茶

让我心算一下，如果串行总共需要 20 分钟，但很显然在烧开水期间，我们可以洗茶壶/洗茶杯/拿茶叶

![image-20210823152742539](https://tva1.sinaimg.cn/large/008i3skNly1gtqr6u2hh4j61fq0e00ua02.jpg)

这样总共需要 16 分钟，节约了 4分钟时间，烧水泡茶尚且如此，在现在高并发的时代，4分钟可以做的事太多了，学会使用 Future 优化程序是必然（**其实优化程序就是寻找关键路径，关键路径找到了，非关键路径的任务通常就可以和关键路径的内容并行执行了**）

```java
@Slf4j
public class MakeTeaExample {

   public static void main(String[] args) throws ExecutionException, InterruptedException {
      ExecutorService executorService = Executors.newFixedThreadPool(2);

      // 创建线程1的FutureTask
      FutureTask<String> ft1 = new FutureTask<String>(new T1Task());
      // 创建线程2的FutureTask
      FutureTask<String> ft2 = new FutureTask<String>(new T2Task());

      executorService.submit(ft1);
      executorService.submit(ft2);

      log.info(ft1.get() + ft2.get());
      log.info("开始泡茶");

      executorService.shutdown();
   }

   static class T1Task implements Callable<String> {

      @Override
      public String call() throws Exception {
         log.info("T1:洗水壶...");
         TimeUnit.SECONDS.sleep(1);

         log.info("T1:烧开水...");
         TimeUnit.SECONDS.sleep(15);

         return "T1:开水已备好";
      }
   }

   static class T2Task implements Callable<String> {
      @Override
      public String call() throws Exception {
         log.info("T2:洗茶壶...");
         TimeUnit.SECONDS.sleep(1);

         log.info("T2:洗茶杯...");
         TimeUnit.SECONDS.sleep(2);

         log.info("T2:拿茶叶...");
         TimeUnit.SECONDS.sleep(1);
         return "T2:福鼎白茶拿到了";
      }
   }
}
```

上面的程序是主线程等待两个 FutureTask 的执行结果，线程1 烧开水时间更长，线程1希望在水烧开的那一刹那就可以拿到茶叶直接泡茶，怎么半呢？



那只需要在线程 1 的FutureTask 中获取 线程 2 FutureTask 的返回结果就可以了，我们稍稍修改一下程序：

```java
@Slf4j
public class MakeTeaExample1 {

   public static void main(String[] args) throws ExecutionException, InterruptedException {
      ExecutorService executorService = Executors.newFixedThreadPool(2);

      // 创建线程2的FutureTask
      FutureTask<String> ft2 = new FutureTask<String>(new T2Task());
      // 创建线程1的FutureTask
      FutureTask<String> ft1 = new FutureTask<String>(new T1Task(ft2));

      executorService.submit(ft1);
      executorService.submit(ft2);

      executorService.shutdown();
   }

   static class T1Task implements Callable<String> {

      private FutureTask<String> ft2;
      public T1Task(FutureTask<String> ft2) {
         this.ft2 = ft2;
      }

      @Override
      public String call() throws Exception {
         log.info("T1:洗水壶...");
         TimeUnit.SECONDS.sleep(1);

         log.info("T1:烧开水...");
         TimeUnit.SECONDS.sleep(15);

         String t2Result = ft2.get();
         log.info("T1 拿到T2的 {}， 开始泡茶", t2Result);
         return "T1: 上茶！！！";
      }
   }

   static class T2Task implements Callable<String> {
      @Override
      public String call() throws Exception {
         log.info("T2:洗茶壶...");
         TimeUnit.SECONDS.sleep(1);

         log.info("T2:洗茶杯...");
         TimeUnit.SECONDS.sleep(2);

         log.info("T2:拿茶叶...");
         TimeUnit.SECONDS.sleep(1);
         return "福鼎白茶";
      }
   }
}
```

![image-20210823152859261](https://tva1.sinaimg.cn/large/008i3skNly1gtqr85w56cj61fo0eu0wf02.jpg)

知道这个变化后我们再回头看 ExecutorService 的三个 submit 方法：

```java
<T> Future<T> submit(Runnable task, T result);
Future<?> submit(Runnable task);
<T> Future<T> submit(Callable<T> task);
```

第一种方法，和我们改造烧水泡茶的程序思维是相似的，可以传进去一个 result，result 相当于主线程和子线程之间的桥梁，通过它主子线程可以共享数据

第二个方法参数是 Runnable 类型参数，即便调用 get() 方法也是返回 null，所以仅是可以用来断言任务已经结束了，类似 Thread.join()

第三个方法参数是 Callable 类型参数，通过get() 方法可以明确获取 call() 方法的返回值



烧水泡茶是非常简单的，如果更复杂业务逻辑，以这种方式使用 Future 必定会带来很大的会乱（程序结束没办法主动通知，Future 的链接和整合都需要手动操作）为了解决这个短板，没错，又是那个男人 Doug Lea, CompletableFuture 工具类在 Java1.8 的版本出现了，搭配 Lambda 的使用，让我们编写异步程序也像写串行代码那样简单，纵享丝滑

接下来我们就了解一下 `CompletableFuture` 的使用

## 灵魂追问

1. 你在日常开发工作中是怎样将整块任务做到分工与协作的呢？有什么基本准则吗？
2. 如何批量的执行异步任务呢？

## 14. CompletableFuture

如果能回答下面脑图里的问题，本节就不用看啦

![image-20210823192846766](https://tva1.sinaimg.cn/large/008i3skNly1gtqy5prfk5j613y0tk40y02.jpg)

### Future 短板

1. **不能手动完成计算:** 

   假设你使用 Future 运行子线程调用远程 API 来获取某款产品的最新价格，服务器由于洪灾宕机了，此时如果你想手动结束计算，而是想返回上次缓存中的价格，这是 Future 做不到的

2. **调用 get() 方法会阻塞程序  ：** 

   Future 不会通知你它的完成，它提供了一个get()方法，程序调用该方法会阻塞直到结果可用为止，没有办法利用回调函数附加到Future，并在Future的结果可用时自动调用它

3. **不能链式执行**

   烧水泡茶中，通过构造函数传参做到多个任务的链式执行，万一有更多的任务，或是任务链的执行顺序有变，对原有程序的影响都是非常大的

4. **整合多个 Future 执行结果方式笨重**

   假设有多个 Future 并行执行，需要在这些任务全部执行完成之后做后续操作，Future 本身是做不到的，需要借助工具类 `Executors` 的方法

   ```java
   <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
   <T> T invokeAny(Collection<? extends Callable<T>> tasks)
   ```

5. **没有异常处理**

   Future 同样没有提供很好的异常处理方案

又是那个男人，并发大师 Doug Lea 忧天下程序员之忧，解天下程序员之困扰，在 Java1.8 版本（Lambda 横空出世）中，新增了一个并发工具类 **CompletableFuture**，它的出现，让人在泡茶过程中，品尝到了不一样的味道......

### 几个重要的`Lambda`函数

**CompletableFuture** 在 Java1.8 的版本中出现，自然也得搭上 Lambda 的顺风车，为了更好的理解 **CompletableFuture**，这里我需要先介绍一下几个 Lambda 函数，我们只需要关注它们的以下几点就可以：

- 参数接受形式
- 返回值形式
- 函数名称

#### 1. Runnable

Runnable 我们已经说过无数次了，无参数，无返回值

```java
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```

#### 2. Function

Function<T, R> 接受一个参数，并且有返回值

```java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
}
```

#### 3. Consumer

Consumer 接受一个参数，没有返回值

```java
@FunctionalInterface
public interface Consumer<T> {   
    void accept(T t);
}
```

#### 4. Supplier

Supplier 没有参数，有一个返回值

```java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}
```

#### 5. BiConsumer

BiConsumer<T, U> 接受两个参数（Bi， 英文单词词根，代表两个的意思），没有返回值

```java
@FunctionalInterface
public interface BiConsumer<T, U> {
    void accept(T t, U u);
```

![image-20210823195137044](https://tva1.sinaimg.cn/large/008i3skNly1gtqytgn1enj61ea0m8whr02.jpg)

为什么要关注这几个函数式接口，因为 **CompletableFuture** 的函数命名以及其作用都是和这几个函数式接口高度相关的，一会你就会发现了

### CompletableFuture

#### 类结构

![image-20210823195258491](https://tva1.sinaimg.cn/large/008i3skNly1gtqyuudw3rj615w0f6my402.jpg)

- 实现了Future接口： 脑补那5个方法
- 实现了 `CompletionStage`接口 : CompletionStage 接口的所有函数都用于描述任务的时序关系，总结起来就是这个样子：

![image-20210823195512438](https://tva1.sinaimg.cn/large/008i3skNly1gtqyx5s13ej61bq0jo76002.jpg)

**CompletableFuture** 既然实现了两个接口，自然也就会实现相应的方法充分利用其接口特性，我们走进它的方法来看一看

#### 串行关系

`then` 直译【然后】，也就是表示下一步，所以通常是一种串行关系体现, then 后面的单词（比如 run /apply/accept）就是上面说的函数式接口中的抽象方法名称了，它的作用和那几个函数式接口的作用是一样一样滴

```java
CompletableFuture<Void> thenRun(Runnable action)
CompletableFuture<Void> thenRunAsync(Runnable action)
CompletableFuture<Void> thenRunAsync(Runnable action, Executor executor)

<U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn)
<U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn)
<U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor)

CompletableFuture<Void> thenAccept(Consumer<? super T> action) 
CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action)
CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action, Executor executor)

<U> CompletableFuture<U> thenCompose(Function<? super T, ? extends CompletionStage<U>> fn)  
<U> CompletableFuture<U> thenComposeAsync(Function<? super T, ? extends CompletionStage<U>> fn)
<U> CompletableFuture<U> thenComposeAsync(Function<? super T, ? extends CompletionStage<U>> fn, Executor executor)
```

#### 聚合 And 关系

`combine... with...` 和 `both...and...` 都是要求两者都满足，也就是 and 的关系了

```java
<U,V> CompletableFuture<V> thenCombine(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)
<U,V> CompletableFuture<V> thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)
<U,V> CompletableFuture<V> thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn, Executor executor)

<U> CompletableFuture<Void> thenAcceptBoth(CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action)
<U> CompletableFuture<Void> thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action)
<U> CompletableFuture<Void> thenAcceptBothAsync( CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action, Executor executor)

CompletableFuture<Void> runAfterBoth(CompletionStage<?> other, Runnable action)
CompletableFuture<Void> runAfterBothAsync(CompletionStage<?> other, Runnable action)
CompletableFuture<Void> runAfterBothAsync(CompletionStage<?> other, Runnable action, Executor executor)
```

#### 聚合 Or 关系

`Either...or...` 表示两者中的一个，自然也就是 Or 的体现了

```java
<U> CompletableFuture<U> applyToEither(CompletionStage<? extends T> other, Function<? super T, U> fn)
<U> CompletableFuture<U> applyToEitherAsync(、CompletionStage<? extends T> other, Function<? super T, U> fn)
<U> CompletableFuture<U> applyToEitherAsync(CompletionStage<? extends T> other, Function<? super T, U> fn, Executor executor)

CompletableFuture<Void> acceptEither(CompletionStage<? extends T> other, Consumer<? super T> action)
CompletableFuture<Void> acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action)
CompletableFuture<Void> acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action, Executor executor)

CompletableFuture<Void> runAfterEither(CompletionStage<?> other, Runnable action)
CompletableFuture<Void> runAfterEitherAsync(CompletionStage<?> other, Runnable action)
CompletableFuture<Void> runAfterEitherAsync(CompletionStage<?> other, Runnable action, Executor executor)
```

#### 异常处理

```java
CompletableFuture<T> exceptionally(Function<Throwable, ? extends T> fn)
CompletableFuture<T> exceptionallyAsync(Function<Throwable, ? extends T> fn)
CompletableFuture<T> exceptionallyAsync(Function<Throwable, ? extends T> fn, Executor executor)

CompletableFuture<T> whenComplete(BiConsumer<? super T, ? super Throwable> action)
CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T, ? super Throwable> action)
CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T, ? super Throwable> action, Executor executor)


<U> CompletableFuture<U> handle(BiFunction<? super T, Throwable, ? extends U> fn)
<U> CompletableFuture<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn)
<U> CompletableFuture<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn, Executor executor)
```

很多同学已经发现了规律：

1. CompletableFuture 提供的所有回调方法都有两个异步（Async）变体，都像这样

   ```java
   // thenApply() 的变体
   <U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn)
   <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn)
   <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor)
   ```

2. 方法的名称也都与前戏中说的函数式接口完全匹配，按照这中规律分类之后，这 50 多个方法看起来是不是很轻松了呢？

#### 演示

##### 1. `runAsync` 进行异步计算

```java
CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(3);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    System.out.println("运行在一个单独的线程当中");
});

future.get();
```

由于使用的是 Runnable 函数式表达式，自然也不会获取到

##### 2. supplyAsync

使用 `runAsync` 是没有返回结果的，我们想获取异步计算的返回结果需要使用 `supplyAsync()` 方法

```java
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                throw new IllegalStateException(e);
            }
            log.info("运行在一个单独的线程当中");
            return "我有返回值";
        });

        log.info(future.get());
```

由于使用的是 Supplier 函数式表达式，自然可以获得返回结果

![image-20210823201535089](https://tva1.sinaimg.cn/large/008i3skNly1gtqzidczqhj61ga090q4u02.jpg)

###### `thenApply`： 从回调中返回结果，并将第一阶段的结果传入第二阶段

对于真正的异步处理，我们希望的是可以通过传入回调函数，在Future 结束时自动调用该回调函数，这样，我们就不用等待结果

```java
CompletableFuture<String> comboText = CompletableFuture.supplyAsync(() -> {
            return "赞";
        })
        .thenApply(first -> {
                    log.info("在看");
                    return first + ", 在看";
        })
        .thenApply(second -> second + ", 转发");

        log.info("三连有没有？");
        log.info(comboText.get());
```

![image-20210823201722417](https://tva1.sinaimg.cn/large/008i3skNly1gtqzk8b9yij61g60caadb02.jpg)

###### `thenAccept`: 如果不需要从回调函数中返回任何结果，那可以使用 thenAccept

```java
        final CompletableFuture<Void> voidCompletableFuture = CompletableFuture.supplyAsync(
                // 模拟远端API调用，这里只返回了一个构造的对象
                () -> Product.builder().id(12345L).name("颈椎/腰椎治疗仪").build())
                .thenAccept(product -> {
                    log.info("获取到远程API产品名称 " + product.getName());
                });
        voidCompletableFuture.get();
```

###### `thenRun`:`thenAccept` 可以从回调函数中获取前序执行的结果，但thenRun 却不可以，因为它的回调函数式表达式定义中没有任何参数

---

我们前面同样说过了，每个提供回调方法的函数都有两个异步（Async）变体，异步就是另外起一个线程

```JAVA
    CompletableFuture<String> stringCompletableFuture = CompletableFuture.supplyAsync(() -> {
        log.info("前序操作");
        return "前需操作结果";
    }).thenApplyAsync(result -> {
        log.info("后续操作");
        return "后续操作结果";
    });
```
###### `thenCompose`

日常的任务中，通常定义的方法都会返回 CompletableFuture 类型，这样会给后续操作留有更多的余地，假如有这样的业务（X呗是不是都有这样的业务呢？）：

```java
//获取用户信息详情
    CompletableFuture<User> getUsersDetail(String userId) {
        return CompletableFuture.supplyAsync(() -> User.builder().id(12345L).name("日拱一兵").build());
    }

//获取用户信用评级
CompletableFuture<Double> getCreditRating(User user) {
    return CompletableFuture.supplyAsync(() -> CreditRating.builder().rating(7.5).build().getRating());
}
```

这时，如果我们还是使用 thenApply() 方法来描述串行关系，返回的结果就会发生 CompletableFuture 的嵌套

```java
    CompletableFuture<CompletableFuture<Double>> result = completableFutureCompose.getUsersDetail(12345L)
            .thenApply(user -> completableFutureCompose.getCreditRating(user));
```
显然这不是我们想要的，如果想“拍平” 返回结果，thenCompose 方法就派上用场了

```java
CompletableFuture<Double> result = completableFutureCompose.getUsersDetail(12345L)
                .thenCompose(user -> completableFutureCompose.getCreditRating(user));
```

这个和 Lambda 的map 和 flatMap 的道理是一样一样滴

###### `thenCombine`  重要

###### 如果要聚合两个独立 Future 的结果，那么 thenCombine 就会派上用场了

```java
    CompletableFuture<Double> weightFuture = CompletableFuture.supplyAsync(() -> 65.0);
    CompletableFuture<Double> heightFuture = CompletableFuture.supplyAsync(() -> 183.8);

    CompletableFuture<Double> combinedFuture = weightFuture
            .thenCombine(heightFuture, (weight, height) -> {
                Double heightInMeter = height/100;
                return weight/(heightInMeter*heightInMeter);
            });

    log.info("身体BMI指标 - " + combinedFuture.get());
```
> 这个就非常适合分批处理，大查询分开查

当然这里多数时处理两个 Future 的关系，如果超过两个Future，如何处理他们的一些聚合关系呢？

###### `allOf | anyOf`

相信你看到方法的签名，你已经明白他的用处了，这里就不再介绍了

```java
static CompletableFuture<Void>     allOf(CompletableFuture<?>... cfs)
static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs)
```

###### `exceptionally`

```java
        Integer age = -1;

        CompletableFuture<String> maturityFuture = CompletableFuture.supplyAsync(() -> {
            if( age < 0 ) {
                throw new IllegalArgumentException("何方神圣？");
            }
            if(age > 18) {
                return "大家都是成年人";
            } else {
                return "未成年禁止入内";
            }
        }).thenApply((str) -> {
            log.info("游戏开始");
            return str;
        }).exceptionally(ex -> {
            log.info("必有蹊跷，来者" + ex.getMessage());
            return "Unknown!";
        });

        log.info(maturityFuture.get());
```

exceptionally 就相当于 catch，出现异常，将会跳过 thenApply 的后续操作，直接捕获异常，进行一场处理

###### `handle`

用多线程，良好的习惯是使用 try/finally 范式，handle 就可以起到 finally 的作用，对上述程序做一个小小的更改， handle 接受两个参数，一个是正常返回值，一个是异常

> **注意：handle的写法也算是范式的一种**

```java
        Integer age = -1;

        CompletableFuture<String> maturityFuture = CompletableFuture.supplyAsync(() -> {
            if( age < 0 ) {
                throw new IllegalArgumentException("何方神圣？");
            }
            if(age > 18) {
                return "大家都是成年人";
            } else {
                return "未成年禁止入内";
            }
        }).thenApply((str) -> {
            log.info("游戏开始");
            return str;
        }).handle((res, ex) -> {
            if(ex != null) {
                log.info("必有蹊跷，来者" + ex.getMessage());
                return "Unknown!";
            }
            return res;
        });

        log.info(maturityFuture.get());
```

---

到这里，关于 `CompletableFuture` 的基本使用你已经了解的差不多了，不知道你是否注意，我们前面说的带有 Sync 的方法是单独起一个线程来执行，但是我们并没有创建线程，这是怎么实现的呢？

细心的朋友如果仔细看每个变种函数的第三个方法也许会发现里面都有一个 Executor 类型的参数，用于指定线程池，因为实际业务中我们是严谨手动创建线程的，这在 我会手动创建线程，为什么要使用线程池?文章中明确说明过；如果没有指定线程池，那自然就会有一个默认的线程池，也就是 `ForkJoinPool`

```java
private static final Executor ASYNC_POOL = USE_COMMON_POOL ?
    ForkJoinPool.commonPool() : new ThreadPerTaskExecutor();
```

ForkJoinPool 的线程数默认是 CPU 的核心数。但是，在前序文章中明确说明过：

> **不要所有业务共用一个线程池**，因为，一旦有任务执行一些很慢的 I/O 操作，就会导致线程池中所有线程都阻塞在 I/O 操作上，从而造成线程饥饿，进而影响整个系统的性能

## 总结

`CompletableFuture` 的方法并没有全部介绍完全，也没必要全部介绍，相信大家按照这个思路来理解 `CompletableFuture` 也不会有什么大问题了，剩下的就交给`实践/时间`以及自己的体会了

## 灵魂追问

1. 听说 ForkJoinPool 线程池效率更高，为什么呢？
2. 如果批量处理异步程序，有什么可用的方案吗？