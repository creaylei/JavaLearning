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

