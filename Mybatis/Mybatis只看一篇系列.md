# Springboot集成Mybatis

> 自己是个渣渣，这个都弄不好  愁人，所以特此记下来  2019.6.5

[![github](https://img.shields.io/badge/github-mybatis-brightgreen.svg)](https://github.com/creaylei/learngit)


## 1.基础版本Mybatis

### 1.创建项目引入依赖

```
创建项目的时候选

Web、 Mybatis、 JDBC、Mysql   (必须)

Lombok、actuator

```



### 2.配置application.yml

```
//这里要配置 mybatis的Mapper包映射

mybatis:
  mapper-locations: classpath:[包名/*Mapper.xml]
  //开启sql日志输出到控制台
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl      
```



### 3.启动类加Dao层的扫描

```java
@SpringBootApplication
@MapperScan("com.example.demo.dao")  //这里面是Dao层的interface
pullic class Application{

}
```



### 4.编写Dao层

```java
//Entity
写一个
@Data
public class BaseEntity{
	//里面写一些通用的属性
		private Long id;

    private Boolean isDelete;

    private Date createTime;

    private Date updateTime;
}

新的实体类进行继承
@Data
public class UserPo extends BaseEntity{

    private String userName;

    private Integer sex;

    private Integer age;
}

//Controller

//Service

//Dao
```



### 5.sql映射

query查询的时候遇到的一些坑

```xml
//1当查询id为1的数据，开始是这样的写法，用的select * 
<select id="queryByPrimary" parameterType="java.lang.Long" resultType="com.example.demo.po.UserPo">
    SELECT * FROM t_user where id = #{id}
</select>

返回
{
    "userName": "null",
    "sex": 0,
    "age": 25,
    "id": 1,
    "isDelete": null,
    "createTime": "null",
    "updateTime": "null"
}

<!--发现没有使用sql映射的，查出来列名和PO类中不一致的属性都是null-->

//2修改
添加 <sql id="Base_Column_List">
        id as id,
        user_name as userName,
        sex as sex,
        age as age,
        is_delete as isDelete,
        create_time as createTime,
        update_time as updateTime
    </sql>

修改查询，映射起来

<select id="queryByPrimary" parameterType="java.lang.Long" resultType="com.example.demo.po.UserPo">
    SELECT
    <include refid="Base_Column_List"/>
    FROM t_user where id = #{id}
</select>

返回
{
    "userName": "zhangsan",
    "sex": 0,
    "age": 25,
    "id": 1,
    "isDelete": false,
    "createTime": "2019-06-05T15:45:28.000+0000",
    "updateTime": "2019-06-05T15:45:28.000+0000"
}
```



### 6.ResultMap映射

```xml
//如果使用Select * 也可以使用ResultMap来进行映射
1. 定义ResultMap，要定义完全
<resultMap id="BaseResultMap" type="com.example.demo.po.UserPo">
        <id column="id" jdbcType="BIGINT" property="id"/>
        <result column="user_name" jdbcType="VARCHAR" property="userName"/>
        <result column="sex" jdbcType="TINYINT" property="sex"/>
        <result column="age" jdbcType="INTEGER" property="age"/>
</resultMap>


此时查询语句
<select id="queryByPrimary" parameterType="java.lang.Long" resultMap="BaseResultMap">
    SELECT *
    FROM t_user where id = #{id}
</select>

返回结果
{
    "userName": "zhanglei",
    "sex": 0,
    "age": 25,
    "id": 1,
    "isDelete": null,
    "createTime": null,
    "updateTime": null
}

注意，这里为什么isDelete，createTime等为Null？  因为在ResultMap中没有进行定义

大牛的一种写法是， Base_Column_List里面不写as，然后返回里面写 resultType，如下
 <resultMap id="BaseResultMap" type="com.shuidihuzhu.cs.biz.po.AgentPo">
    <id column="id" jdbcType="BIGINT" property="id" />
    <result column="agent_id" jdbcType="BIGINT" property="agentId" />
    <result column="account_name" jdbcType="VARCHAR" property="accountName" />
    <result column="nick_name" jdbcType="VARCHAR" property="nickName" />
    <result column="real_name" jdbcType="VARCHAR" property="realName" />
    <result column="email" jdbcType="VARCHAR" property="email" />
    <result column="status" jdbcType="TINYINT" property="status" />
    <result column="is_delete" jdbcType="BIT" property="isDelete" />
    <result column="create_time" jdbcType="TIMESTAMP" property="createTime" />
    <result column="update_time" jdbcType="TIMESTAMP" property="updateTime" />
  </resultMap>
  <sql id="Base_Column_List">
    id, agent_id, nick_name, real_name,email,status, is_delete, create_time, update_time
  </sql>
```

ResultMap中的映射关系

```xml
 <result property="FLD_NUMBER" column="FLD_NUMBER"  javaType="double" jdbcType="NUMERIC"/>  
  <result property="FLD_VARCHAR" column="FLD_VARCHAR" javaType="string" jdbcType="VARCHAR"/>  
  <result property="FLD_DATE" column="FLD_DATE" javaType="java.sql.Date" jdbcType="DATE"/>  
  <result property="FLD_INTEGER" column="FLD_INTEGER"  javaType="int" jdbcType="INTEGER"/>  
  <result property="FLD_DOUBLE" column="FLD_DOUBLE"  javaType="double" jdbcType="DOUBLE"/>  
  <result property="FLD_LONG" column="FLD_LONG"  javaType="long" jdbcType="INTEGER"/>  
  <result property="FLD_CHAR" column="FLD_CHAR"  javaType="string" jdbcType="CHAR"/>  
  <result property="FLD_BLOB" column="FLD_BLOB"  javaType="[B" jdbcType="BLOB" />  
  <result property="FLD_CLOB" column="FLD_CLOB"  javaType="string" jdbcType="CLOB"/>  
  <result property="FLD_FLOAT" column="FLD_FLOAT"  javaType="float" jdbcType="FLOAT"/>  
  <result property="FLD_TIMESTAMP" column="FLD_TIMESTAMP"  javaType="java.sql.Timestamp" jdbcType="TIMESTAMP"/>  
```

至此，我们的Mybatis基础版就可以搭建完成了
