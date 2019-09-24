# Java反射

> https://www.jianshu.com/p/10c29883eac1

## 什么是反射

- Java反射机制是在运行状态中
- 对于**任意一个类**，都能知道这个类的所以**属性和方法**；
- 对于**任何一个对象**，都能够调用它的任何一个**方法和属性**；
- 这样**动态获取**新的以及**动态调用**对象方法的功能就叫做**反射**。

### Class类：一切反射的基础

## 1 获取Class类

> 三种方法

```java
Class c1 = Test.class; //这说明任何一个类都有一个隐含的静态成员变量class，这种方式是通过获取类的静态成员变量class得到的()
Class c2 = test.getClass();// test是Test类的一个对象，这种方式是通过一个类的对象的getClass()方法获得的 (对于基本类型无法使用这种方法)
Class c3 = Class.forName("com.catchu.me.reflect.Test"); //这种方法是Class类调用forName方法，通过一个类的全量限定名获得（基本类型无法使用此方法）
```

## 2 获取类的细节

### 2. 获取Field

> 通过Field你可以访问给定对象的类变量，包括获取**变量的类型、修饰符、注解、变量名、变量的值或者重新设置变量值**，即使变量是private的。

四种方法获取给定类的Field

1. [`getDeclaredField(String name)`](https://docs.oracle.com/javase/8/docs/api/java/lang/Class.html#getDeclaredField-java.lang.String-)
   获取指定的变量（只要是声明的变量都能获得，包括private）
2. [`getField(String name)`](https://docs.oracle.com/javase/8/docs/api/java/lang/Class.html#getField-java.lang.String-) 
   获取指定的变量（只能获得public的）
3. [`getDeclaredFields()`](https://docs.oracle.com/javase/8/docs/api/java/lang/Class.html#getDeclaredFields--) 
   获取所有声明的变量（包括private）
4. [`getFields()`](https://docs.oracle.com/javase/8/docs/api/java/lang/Class.html#getFields--) 
   获取所有的public变量

### 2.2 获取Method

Class依然提供了4种方法获取Method:

- [`getDeclaredMethod(String name, Class... parameterTypes)`](https://docs.oracle.com/javase/8/docs/api/java/lang/Class.html#getDeclaredMethod-java.lang.String-java.lang.Class...-)
  根据方法名获得指定的方法， 参数name为方法名，参数parameterTypes为方法的参数类型，如 getDeclaredMethod(“eat”, String.class)
- [`getMethod(String name, Class... parameterTypes)`](https://docs.oracle.com/javase/8/docs/api/java/lang/Class.html#getMethod-java.lang.String-java.lang.Class...-)
  根据方法名获取指定的public方法，其它同上
- [`getDeclaredMethods()`](https://docs.oracle.com/javase/8/docs/api/java/lang/Class.html#getDeclaredMethods--)
  获取所有声明的方法
- [`getMethods()`](https://docs.oracle.com/javase/8/docs/api/java/lang/Class.html#getMethods--)
  获取所有的public方法

**获取方法返回类型**
[`getReturnType()`](https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/Method.html#getReturnType--) 获取目标方法返回类型对应的Class对象
[`getGenericReturnType()`](https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/Method.html#getGenericReturnType--) 获取目标方法返回类型对应的Type对象

**获取方法参数类型**
[`getParameterTypes()`](https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/Method.html#getParameterTypes--) 获取目标方法各参数类型对应的Class对象
[`getGenericParameterTypes()`](https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/Method.html#getGenericParameterTypes--) 获取目标方法各参数类型对应的Type对象
返回值为数组，它俩区别同上 “方法返回类型的区别” 。

**获取方法声明抛出的异常的类型**
[`getExceptionTypes()`](https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/Method.html#getExceptionTypes--) 获取目标方法抛出的异常类型对应的Class对象
[`getGenericExceptionTypes()`](https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/Method.html#getGenericExceptionTypes--) 获取目标方法抛出的异常类型对应的Type对象

**获取方法参数名称**
.class文件中默认不存储方法参数名称，如果想要获取方法参数名称，需要在编译的时候加上`-parameters`参数。(构造方法的参数获取方法同样)

```java
//这里的m可以是普通方法Method，也可以是构造方法Constructor
//获取方法所有参数
Parameter[] params = m.getParameters();
for (int i = 0; i < params.length; i++) {
    Parameter p = params[i];
    p.getType();   //获取参数类型
    p.getName();  //获取参数名称，如果编译时未加上`-parameters`，返回的名称形如`argX`, X为参数在方法声明中的位置，从0开始
    p.getModifiers(); //获取参数修饰符
    p.isNamePresent();  //.class文件中是否保存参数名称, 编译时加上`-parameters`返回true,反之flase
}
```

**获取方法修饰符**
方法与Filed等类似  `method.getModifiers`

### **通过反射调用方法**: 重要

反射通过Method的[`invoke()`](https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/Method.html#invoke-java.lang.Object-java.lang.Object...-)方法来调用目标方法。第一个参数为需要调用的**目标类对象**，如果方法为static的，则该参数为null。后面的参数都为目标方法的参数值，顺序与目标方法声明中的参数顺序一致。

```java
public native Object invoke(Object obj, Object... args)
            throws IllegalAccessException, IllegalArgumentException, InvocationTargetException
```

## 3 反射缺点

> - **性能问题**。因为反射是在**运行时而不是在编译时**，所有不会利用到编译优化，同时因为是动态生成，因此，反射操作的效率要比那些非反射操作低得多。
> - **安全问题**。使用反射技术要求程序必须在一个没有安全限制的环境中运行。如果一个程序必须在有安全限制的环境中运行，如Applet，那么这就是个问题了。
> - **代码问题**。由于反射允许代码执行一些在正常情况下不被允许的操作（比如访问私有的属性和方法），所以使用反射可能会导致意料之外的副作用－－代码有功能上的错误，降低可移植性。反射代码破坏了抽象性，因此当平台发生改变的时候，代码的行为就有可能也随着变化。