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

