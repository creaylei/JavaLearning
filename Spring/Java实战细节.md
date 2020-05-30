# Java相关知识

## 1. Stream操作集合

[Stream只看一篇](https://mp.weixin.qq.com/s/rJFbM0pUmK0PR_ldKKGTTg)

## 2. Assert使用

自己使用

```java
@PostMapping("/adduser")
    public int addUserSelective(String userName, Integer sex, Integer age) {
        Assert.notNull(userName,"username不能为空");
        UserPo userPo = new UserPo();
        userPo.setUserName(userName);
        userPo.setSex(sex);
        userPo.setAge(age);
        return userService.addUserSelective(userPo);
    }
```

传参为  sex, age

```java
{
    "timestamp": "2019-10-14T03:33:56.884+0000",
    "status": 500,
    "error": "Internal Server Error",
    "message": "username不能为空",
    "path": "/redis/demo/user/adduser"
}
```

## 3. 快速搭建一个Spring-Boot

[快速搭建](https://mp.weixin.qq.com/s/AU8Ccg5sgAXJGT-bQHayjw) 涉及点比较多，启步

## 4. SpringBoot能遇到的所有开发细节

http://cmsblogs.com/?p=3821

包括：

- 权限设计：shiro
- 持久层集成
- 消息队列的应用
- actutor使用介绍
- 本地锁和分布式锁
- 数据验证 @Valid @Validated
- 文件上传

## 5. DTO-PO转化抽象

> 假设有UserPO 和 UserDTO, 要实现相互转化
>
> 我们就来逐步升级打怪吧

假设我们定义了一个添加用户的方法

```java
@PostMapping
    public User addUser(UserDTO userDTO){
        UserPO userPO = new User();
        userPO.setUsername(userDTO.getUsername());
        userPO.setAge(userDTO.getAge());

        return userService.addUser(userPO);
    }
```

### 1. 第一级

```java
//如同上面所使用的
		UserPO userPO = new UserPO();
    userPO.setUsername(userDTO.getUsername());
    userPO.setAge(userDTO.getAge());
```

### 2. 第二级

```java
//使用BeanUtils.copyProperties
UserPO userPO = new UserPO();
BeanUtils.copyProperties(userDTO,userPO);
```

### 3. 第三级

```java
//自定义converter
@PostMapping
 public UserPO addUser(UserDTO userDTO){
         UserPO user = convertFor(userInputDTO);

         return userService.addUser(user);
 }

//自定义转换器
 private UserPO convertFor(UserDTO userDTO){

         UserPO user = new User();
         BeanUtils.copyProperties(userInputDTO,user);
         return user;
 }
```

### 4. 第四级

```
//抽象出接口
public interface converter<PO, DTO> {
	DTO convertToDTO(PO s);
	
	PO convertToPO(DTO t);
	
	List<DTO> convertToDTOList(List<PO> sList);
	
	List<PO> convertToPOList(List<DTO> tList);
}

//实现类
public UserConverter implements converter {
	
	//这里我们只展示第一个方法
	@Override
	public UserPO convert(UserDTO userDTO) {
		UserPO userPO = new UserPO();
		BeanUtils.copyProperties(userDTO,userPO);
		return userPO;
	}
}
```

### 5. 第五级

```java
//在po或者dto类中实现接口和内部类
@Setter
@Getter
public class UserDTO {
    @NotNull
    private String username;
    @NotNull
    private int age;

    public User convertToUser(){
        UserDTOConvert userDTOConvert = new UserDTOConvert();
        User convert = userDTOConvert.convert(this);
        return convert;
    }

    public UserDTO convertFor(User user){
        UserDTOConvert userDTOConvert = new UserDTOConvert();
        UserDTO convert = userDTOConvert.reverse().convert(user);
        return convert;
    }

    private static class UserDTOConvert extends Converter<UserDTO, User> {
        @Override
        protected User doForward(UserDTO userDTO) {
            User user = new User();
            BeanUtils.copyProperties(userDTO,user);
            return user;
        }

        @Override
            protected UserDTO doBackward(User user) {
                    UserDTO userDTO = new UserDTO();
                    BeanUtils.copyProperties(user,userDTO);
                    return userDTO;
            }
    }

}
```

然后代码就变化为了

然后api中的转化则由:

```
User user = new UserDTOConvert().convert(userInputDTO);
```

变成了:

```
User user = userDTO.convertToUser();
```

## 6 添加时间转换到SpringMVC

> 这部分可以归到自定义Spring MVC配置里面 http://springboot.javaboy.org/2019/0912/springmvc-config

场景： 开发中前端常常传的日期类型是String的，而后端在存入数据库中的时候，是Date类型的，这时候我们就可以在项目中添加converter，具体如下

### 自定义converter

```java
import org.apache.commons.lang.StringUtils;
import org.springframework.core.convert.converter.Converter;

import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;

public class DateConverter implements Converter<String, Date> {

    private SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd");

    @Override
    public Date convert(String s) {
        if(StringUtils.isEmpty(s)) {
            return null;
        }
        try {
            return simpleDateFormat.parse(s);
        } catch (ParseException e) {
            e.printStackTrace();
        }
        return null;
    }
}

```

### 加入到SpringMVC中去

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.format.FormatterRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(new DateConverter());
    }
}
```

这样在前端传入String的时候，会默认的转为 Date类型

### 反向怎么弄呢？

如果查询的时候，直接从数据库中出来的是Timestamp的，怎么能统一转成String呢

#### 方法一

我们在使用@RestController的时候，会将Date类型的数据映射为TIMESTAMP的

解决方案：在 application.yml 配置文件中配置

```xml
spring.jackson.date-format=yyyy-MM-dd HH:mm:ss
spring.jackson.time-zone=GMT+8
```

## 7. 自定义MVC组件

> `WebMvcConfigurerAdapter`   `WebMvcConfigurationSupport`     `WebMvcConfigurer`

`WebMvcConfigurerAdapter` 已经淘汰，替代类是  `WebMvcConfigurationSupport`  比adapter的方法更多，更有用。    

两种方法

- 继承`WebMvcConfigurationSupport` 类
- 实现`WebMvcConfigurer` 接口

## 8. 字符串类

### 1. split方法

```java
				String str = "1,2,,3,,";
        String[] strArr = str.split(",");
        for (int i = 0; i < strArr.length; i++) {
            System.out.println("第"+i+"个字符："+strArr[i]);
        }

-----------------------------输出--------------------------------
第0个字符：1
第1个字符：2
第2个字符：
第3个字符：3
```

可以k按到，这里str后面是有2个“,”的，按理说解析出来的数组长度应该是6，但是实际是4。

**那么看看源码**

```java
		public String[] split(String regex) {
        return split(regex, 0);
    }
```

这里调用的是 split(regex, limit)

> regex是分隔符， limit 控制所需要分隔的次数。   如果该限制n大于0， 则模式最多可应用n-1次，数组的长度不大于n，而且数组最后一项会包含所有超出最后匹配的定界符的输入。也可以理解为最后输出的数组长度

#### 案例

```java
1. 			String str = "1,2,,3,,";
        String[] strArr = str.split(",",1);
        for (int i = 0; i < strArr.length; i++) {
            System.out.println("第"+i+"个字符："+strArr[i]);
        }
        ==================输       出=====================
        第0个字符：1,2,,3,,       这里输出的数组长度为1，即被切分 1-1 =0 次
        
2. 			String str = "1,2,,3,,";
        String[] strArr = str.split(",",2);
        for (int i = 0; i < strArr.length; i++) {
            System.out.println("第"+i+"个字符："+strArr[i]);
        }
        ==================输       出=====================
        第0个字符：1
				第1个字符：2,,3,,        输出两个数组，以第一个, 开始切分
				
3. 			String str = "1,2,,3,,";
        String[] strArr = str.split(",");
        for (int i = 0; i < strArr.length; i++) {
            System.out.println("第"+i+"个字符："+strArr[i]);
        }
        ==================输       出=====================
        第0个字符：1
				第1个字符：2
				第2个字符：
				第3个字符：3              这里3后面的切分，由于是空的，不予显示
          
4. 			limit设置为非正，可以全部解析
        String str = "1,2,,3,,";
        String[] strArr = str.split(",", -1);
        for (int i = 0; i < strArr.length; i++) {
            System.out.println("第"+i+"个字符："+strArr[i]);
        }
				==================输       出=====================
        第0个字符：1
				第1个字符：2
				第2个字符：
				第3个字符：3
				第4个字符：
				第5个字符：
```

注意：
1、如果用“.”作为分隔的话,必须是如下写法,String.split("\\."),这样才能正确的分隔开,不能用String.split(".");
2、如果用“|”作为分隔的话,必须是如下写法,String.split("\\|"),这样才能正确的分隔开,不能用String.split("|");

## 9 set 和 get方法

项目开发中有个涉及全局的需求。即 将所有的手机号存到mysql的时候加密，给前端的时候打码。 思考再三觉得将set或者get方法重写较好。

下面三种情况都要调用一下get()方法

1. 从Mybatis查询，映射到实体类的时候，会调用po类的get和set方法
2. BeanUtils.copyProperties（A， B）  将A的属性拷贝到B的时候，会调用A的get，B的set
3. @RestController  在从后端返回到前端的时候，会再次调用get() 方法

> 对于一些枚举类，我们可以在DTO类里面，加上desc，   get方法里转换就可以

## 10 Excel相关

https://blog.csdn.net/u011794238/article/details/46276551