# Fastjson

> 很常用的json库

## 1. 优点

- 速度快，目前最快的Json库 (2019年)
- 使用广泛，阿里大规模使用，广泛部署
- 测试完备，fastjson有非常多的testcase，在1.2.11版本中，testcase超过3321个。每次发布都会进行回归测试，保证质量稳定
- 使用简单、功能完备

## 2. 使用

1. 引入pom，建议使用最新的

   ```
   <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>x.x.x</version>
   </dependency>
   ```

2. 主要API

   - 对象转jsonString

   ```java
   String str = JSON.toJSONString(obj)
   ```

   - JsonString转对象

   ```
   VO vo = JSON.parseObject("...", VO.class);
   ```

   - 泛型序列化

   ```
   List<VO> list = JSON.parseObject("...", new TypeReference<List<VO>>() {});
   ```

3. 处理日期

`JSON.toJSONStringWithDateFormat(date, "yyyy-MM-dd HH:mm:ss.SSS")`

## 3. FastJson定制序列化

fastjson支持多种方式定制序列化。

- 通过@JSONField定制序列化
- 通过@JSONType定制序列化
- 通过SerializeFilter定制序列化
- 通过ParseProcess定制反序列化

### 1. 使用@JSONField配置

1. JSONField注解介绍

```
package com.alibaba.fastjson.annotation;

public @interface JSONField {
    // 配置序列化和反序列化的顺序，1.1.42版本之后才支持,即序列化后字段的排序
    int ordinal() default 0;

     // 指定字段的名称
    String name() default "";

    // 指定字段的格式，对日期格式有用
    String format() default "";

    // 是否序列化
    boolean serialize() default true;

    // 是否反序列化
    boolean deserialize() default true;
}
```

2. 使用方式

可以把@JSONField配置在字段或者getter/setter方法上，例如：

```java
//字段上
public class VO {
     @JSONField(name="ID")
     private int id;

     @JSONField(name="birthday",format="yyyy-MM-dd")
     public Date date;
}

//配置在Getter/Setter上
public class VO {
    private int id;

    @JSONField(name="ID")
    public int getId() { return id;}

    @JSONField(name="ID")
    public void setId(int id) {this.id = id;}
}
```

3. 使用format配置日期格式化

```
 public class A {
      // 配置date序列化和反序列使用yyyyMMdd日期格式
      @JSONField(format="yyyyMMdd")
      public Date date;
 }
```

4. **使用serialize/deserialize指定字段不序列化**

```java
public class A {
      @JSONField(serialize=false)
      public Date date;
 }

 public class A {
      @JSONField(deserialize=false)
      public Date date;
 }
```

5. **使用ordinal指定字段的顺序**

缺省Fastjson序列化一个java bean，是根据fieldName的字母序进行序列化的，你可以通过ordinal指定字段的顺序。这个特性需要1.1.42以上版本。

```java
public static class VO {
    @JSONField(ordinal = 3)
    private int f0;

    @JSONField(ordinal = 2)
    private int f1;

    @JSONField(ordinal = 1)
    private int f2;
}
```

[更多配置方式]([file:///private/var/folders/_m/b_q50wf16bs99xcklnhx6spm0000gn/T/WizNote/0c5cc56b-9426-47b7-aeb6-e110dc11e993/index.html](file:///private/var/folders/_m/b_q50wf16bs99xcklnhx6spm0000gn/T/WizNote/0c5cc56b-9426-47b7-aeb6-e110dc11e993/index.html))

