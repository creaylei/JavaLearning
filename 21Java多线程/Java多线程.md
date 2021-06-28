# Java多线程

[Toc]

---

## 1. 多线程基础

### 1. 线程和进程的区别

### 2. 线程的三种创建方式

- **继承`Thread` 类**， 重写 `run()` 方法， 调用`start()` 方法启动多线程。  调用后何时执行，由CPU决定

  ```java
  class MyThread extends Thread{
  		public void run() {
  				//方法体
  		}
  		
  		public static void main(String[] args) {
  			MyThread t = new MyThread();
  			t.start();
  		}
  }
  ```

  `Thread` 接口实际也实现了 `Runnable()` 接口

- **实现 `Runnabel` 接口**， 重写 `run()` 方法。 执行线程需要丢入 Runnable() 接口实现类

  ```java
  class MyThread implents Runnable{
  		public void run() {
  				//方法体
  		}
  		
  		public static void main(String[] args) {
  			MyThread t = new MyThread();
  			new Thread(t).start();
  		}
  }
  ```

  

- **实现 `Callable` 接口**，重写 `call()` 方法。

  优点： 1. 可以获取结果  2. 可以捕获异常 。   但是提交方法是使用线程池方式

  ```java
  class MyThread implents Callable{
  		public boolean call() {
  				//方法体
  		}
  		
  		public static void main(String[] args) {
  			MyThread t = new MyThread();
  			//创建线程池
  			ExecutorService service = Executors.newFixedThreadPool(1);
  			Future<Boolean> future =  service.submit(t);
        boolean result = future.get();
        service.shutdown();
  		}
  }
  ```

### 3. 补充一下Lambda表达式

函数式接口：只含有一个方法的接口

对函数式接口，可以同lambda表达式简化

```java
package org.javaboy.security.config;

public class TestLambda {

    //2 静态内部类
    static class Like2 implements ILike{
        @Override
        public void lambda() {
            System.out.println("i like lambda2");
        }
    }

    public static void main(String[] args) {
        ILike like = new Like();
        like.lambda();

        like = new Like2();
        like.lambda();

        //3 局部类：直接写在代码里的类
        class Like3 implements ILike{
            @Override
            public void lambda() {
                System.out.println("i like lambda3");
            }
        }

        like = new Like3();
        like.lambda();

        //4 匿名内部类： 没有类的名称，必须借助接口或者父类名称
        like = new ILike() {
            @Override
            public void lambda() {
                System.out.println("i like lambda4");
            }
        };
        like.lambda();

        //5 开始用lambda替换
        like = ()->{
            System.out.println("i like lambda5");
        };
        like.lambda();

        like = ()-> System.out.println("i like lambda6");
        like.lambda();

    }

}

interface ILike{
    void lambda();
}

//1 内部类
class Like implements ILike{
    @Override
    public void lambda() {
        System.out.println("i like lambda");
    }
}
```

注意其中各种内部类的表述，并且怎么用lambda表达式替换的

---

### 4. 线程的生命周期

![image-20210627214445532](https://tva1.sinaimg.cn/large/008i3skNly1grx5rmbqv5j31800j4qcx.jpg)



#### 4.0 线程的状态

```java
		//Thread类的内部状态
		public static enum State {
        NEW,
        RUNNABLE,
        BLOCKED,
        WAITING,
        TIMED_WAITING,   //等待另一个线程执行完成
        TERMINATED;
    }
```

**新建、就绪、运行、阻塞、销毁**。  

**`new` 一个的时候就新生了， 然后调用`start()`, 或者`submit` 就提交就绪，是否执行需要看CPU调度**

调用 Thread.getState() 获取当前状态

#### 4.1 线程休眠Sleep()

`sleep` 时间到达后进入`就绪`状态，可以模拟网络延时，倒计时

每个对象都有锁，`sleep` 不会释放锁

#### 4.2 线程礼让`yield()`

礼让是让当前的线程从`运行`进入`就绪`状态，不一定会成功，看CPU心情

线程暂停，但是不阻塞

#### 4.3 强制执行 `join()`

简单来讲就是插队， 强制当前线程执行完，再执行其他线程

#### 4.4 线程优先级

`getPriority()` 和`setPriority()` ， 线程优先级从 0 -10

JAVA提供了一个线程调度器，用来监控所有进入就绪状态的线程。 优先级高的线程将大概率被优先调度

**优先级高低，只是概率上的高低，不是绝对的**

#### 4.5 守护线程 (Daemon)

线程分为`守护线程` 和 `用户线程`

`setDaemon(true)`  true表示【守护线程】，否则是【用户线程】

虚拟机不管【守护线程】是否结束，但必须确保【用户线程】执行完毕

---

### 5. 线程同步机制

涉及场景：多个线程操作同一个资源

线程同步：其实就是一种等待机制

同步条件：队列 + 锁 🔐

