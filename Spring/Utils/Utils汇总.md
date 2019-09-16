# Utils

# 1. DateUtils

> java自带的 SimpleDateFormat，在多线程的情况下并不安全，因为它不是同步的。 这里原理不多讲，详情请看 [Java SimpleDateFormat 没那么简单](https://mp.weixin.qq.com/s/BeaXocQeUfcl0M5E9GjKWw), 这里我们只讲工具的使用

既然SimpleDateFormat不安全，我们还是用更安全的东西

1. Java8线程安全的时间日期API

   非常值得学习这些类的用法，包括 DateTimeFormatter、OffsetDateTime、ZonedDateTime、LocalDateTime、LocalDate 和 LocalTime。

   ```java
   //使用新API后的代码
   public class DateUtilsJava8 {
       public static final DateTimeFormatter DATE_TIME_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd");
       private DateUtilsJava8() {}
       public static LocalDate parse(String target) {
           return LocalDate.parse(target, DATE_TIME_FORMATTER);
       }
       public static String format(LocalDate target) {
           return target.format(DATE_TIME_FORMATTER);
       }
   }	
   ```

2. jodaTime的使用(更推荐)

> 看别人的例子使用过jodaMoney，这里讲讲jodaTime，更安全，更方便

```java
DateTime dateTime = new DateTime();         获取当前时间

//获取linux时间戳   
dateTime.getMillis()

//常用方法来了
1. DateTime 转 Date类
Date date = dateTime.toDate();

//DateTime 解析年月日
        DateTimeFormatter dateTimeFormat = DateTimeFormat.forPattern("yyyy年MM月dd日 HH:mm:ss");
        System.out.println(dateTimeFormat.print(dateTime.getMillis()));

        DateTimeFormatter format1 = DateTimeFormat.forPattern("yyyy.MM.dd HH:mm:ss");
        System.out.println(format1.print(dateTime.getMillis()));


        DateTimeFormatter format2 = DateTimeFormat.forPattern("yyyy-MM-dd hh:mm:ss");
        System.out.println(format2.print(dateTime.getMillis()));

        DateTime dateTime1 = format1.parseDateTime("2019.07.08 12:00:00");
        System.out.println(dateTime1);
```

# 2. 操作list的stream

> Stream的原理：将要处理的元素看做一种流，流在管道中传输，并且可以在管道的节点上处理，包括过滤筛选、去重、排序、聚合等。元素流在管道中经过中间操作的处理，最后由最终操作得到前面处理的结果。

集合有两种方式生成流：

- stream() − 为集合创建串行流
- parallelStream() - 为集合创建并行流

中间操作主要有以下方法（此类型方法返回的都是Stream）：map (mapToInt, flatMap 等)、 filter、 distinct、 sorted、 peek、 limit、 skip、 parallel、 sequential、 unordered

> 中间操作可以理解为处理list

终止操作主要有以下方法：forEach、 forEachOrdered、 toArray、 reduce、 collect、 min、 max、 count、 anyMatch、 allMatch、 noneMatch、 findFirst、 findAny、 iterator

> 终止操作如输出，给出结果等



# 3. 重构

1. 使用工具函数

```java
不完善的写法
thisName!=null && thisName.equals(name)

完善的写法
(thisName == name) || (thisName != null && thisName.equals(name));

建议方案
Objects.equals(name, thisName);
```

2. 集合判空

```
CollectionUtils.isEmpty(list)
```

3. 尽量避免不必要的空指针判断，
   1. **调用函数保证参数不为空，被调用函数尽量避免不必要的空指针判断**
   2. **被调用函数保证返回不为空,调用函数尽量避免不必要的空指针判断**
4. 内部函数尽量使用基本类型

```
private double calculate(double price, int number) {
    return price * number;
}
```

5. 内部函数返回值也使用基本类型

> **11.3 主要收益**
>
> - 内部函数尽量使用基础类型，避免了隐式封装类型的打包和拆包；
> - 内部函数参数使用基础类型，用语法上避免了内部函数的参数空指针判断；
> - 内部函数返回值使用基础类型，用语法上避免了调用函数的返回值空指针判断。