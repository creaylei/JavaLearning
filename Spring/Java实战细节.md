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

