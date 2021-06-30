# Java并发编程的艺术(二)——基础

[toc]

# 四、JAVA并发编程的基础

## 4.1 线程

线程：运行一个程序会创建一个进程。线程是轻量级进程，是现代操作系统调度的最小单元。

为什么要使用多线程：

1. 更多处理器核心
2. 更快响应时间
3. 更好的编程模型

### 线程优先级

多线程可以通过 priority 来控制优先级， 范围从 1~10。默认是5，优先级高的线程分配的时间片多。

### 线程的状态

- 新建 new     ：初始状态，还没调用start
- 运行 Runnable ： 就绪 或者运行
- 阻塞 Blocked：   线程阻塞于锁
- 等待 Waiting：  等待状态，表示当前线程需要等待其他线程，做出通知动作( 通知或中断 )
- 超时等待  Time-Waiting ： 他可以在指定的时间返回，不同于 Wating
- 终止  Terminated ：终止状态

### 一个排查实例

 

```
public class ThreadState {
    public static void main(String[] args) {
        new Thread(new TimeWaiting(), "TimeWaiting Thread").start();
        new Thread(new Waiting(), "Waiting Thread").start();
        //使用两个Blocked线程，一个获取锁成功，一个获取索失败
        new Thread(new Blocked(), "Blocked Thread——1").start();
        new Thread(new Blocked(), "Blocked Thread——2").start();
    }
    //不断进行睡眠
    static class TimeWaiting implements Runnable{
        @Override
        public void run() {
            while (true) {
                SleepUtils.second(100);
            }
        }
    }
    //该线程在Waiting.class等待
    static class Waiting implements Runnable{
        @Override
        public void run() {
            while (true) {
                synchronized (Waiting.class) {
                    try {
                        Waiting.class.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
        //该线程在Blocked.class实例上加锁后，不会释放锁
    static class Blocked implements Runnable{
        @Override
        public void run() {
            synchronized (Blocked.class) {
                while (true) {
                    SleepUtils.second(100);
                }
            }
        }
    }
}
//此外
public class SleepUtils {
    public static final void second(long seconds) {
        try {
            TimeUnit.SECONDS.sleep(seconds);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

运行该程序，打开终端  键入 “jps”

![image-20200619090355203](https://tva1.sinaimg.cn/large/007S8ZIlly1gfxbn0hijpj30rk07qadr.jpg)

找到 `ThreadState`  进程id 34316， 接着输入  jstack 34316

![image-20200619092722001](https://tva1.sinaimg.cn/large/007S8ZIlly1gfxcbdw8k1j31b20u07wj.jpg)

## 4.2 线程的启动和终止

启动线程： start()方法启动，该方法的含义是，如果当前线程(即Parent线程)同步告诉JVM，只要线程规划器空闲，应立即启动调用start()方法的线程。

启动一个线程前，最好给线程取个名字，这样用jstack排查问题的时候比较方便定位。

### 理解中断

中断可以理解为线程的一个标识位属性，表示一个运行中的线程是否被其他线程进行了中断操作。 中断好比其他线程给该线程打招呼，通过调用该线程的 interrupt()实现。

线程可以通过方法 `isInterrupt()` ，判断自己是否被中断。 也可调用静态方法 Thread.interrupt() 来进行**修复**，如果线程已经终止，会直接返回false。

### 过期的 suspend()  resume() 和 stop()

暂停、恢复和停止。 

这三个API是过期的，不建议使用，以suspend()，会持有已占有的资源(如锁)进入睡眠状态，会引起死锁。    stop在终止的时候，也不能保证资源被释放。 所以不建议使用。

使用 等待/通知 来代替。

## 4.3 线程间通信