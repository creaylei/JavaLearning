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



## 2.升级-分页和遍历集合

### 1. 分页查询的构建

> 网页上常见的分页有两种
>
> - 百度的那种，下面是页数pageNum，然后每页显示pageSize个数据的
> - 信息流方式，先显示pageSize个，然后往下滚动继续给信息的，也是分页

1. 引入依赖

```java
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper</artifactId>
    <version>4.1.6</version>
</dependency>
```

2. 两种使用方式

- 第一种，直接通过RowBounds参数完成分页

```java
List<Student> list = studentMapper.find(new RowBounds(0, 10));
Page page = ((Page) list;
```

- 第二种，PageHelper.startPage()静态方法

```
 //获取第1页，10条内容，默认查询总数count
    PageHelper.startPage(1, 10);
//紧跟着的第一个select方法会被分页
    List<Country> list = studentMapper.find();
    Page page = ((Page) list;
```

注: 返回结果list，已经是Page对象，Page对象是一个arrayList

原理: 使用ThreadLocal来传递和保存Page对象，每次查询，都需要单独设置PageHelper.startPage()方法。

3. 自己使用

> 自己使用的时候，要自己定义分页查询的QueryDTO，该DTO中有， pageSize和pageNum字段

```java
PageInfo<OperationLogDto> pageInfo = operationLogService.queryOperationLogByPage(operationLogQueryDto);

        PageDto<OperationLogDto> pageDto = new PageDto<>();
        pageDto.setList(pageInfo.getList());
        pageDto.setTotal(pageInfo.getTotal());
        pageDto.setPageSize(operationLogQueryDto.getPageSize());
        pageDto.setPageNum(operationLogQueryDto.getPageNum());


其中queryOperationLogByPage方法为

Page<OperationLogDto> page = PageHelper.startPage(operationLogQueryDto.getPageNum(), operationLogQueryDto.getPageSize());
        List<OperationLogPo> operationLogPoList = operationLogDao.queryOperationLogByPage(operationLogQueryDto);
        PageInfo<OperationLogDto> pageInfo = new PageInfo<OperationLogDto>(page.getResult());

```

```xml
Mapper.xml文件中的
<select id="queryOperationLogByPage" resultType="com.shuidihuzhu.cs.robot.po.OperationLogPo">
        SELECT <include refid="Base_Column_List"/> FROM cs_robot_operation_log
        <where>
            <if test="operationLogQueryDto.operateType !=null and operationLogQueryDto.operateType !=''">and operate_type=#{operationLogQueryDto.operateType}</if>
            <if test="operationLogQueryDto.title !=null and operationLogQueryDto.title !=''">and title = #{operationLogQueryDto.title}</if>
            <if test="operationLogQueryDto.requestUri !=null and operationLogQueryDto.requestUri !='' ">and request_uri = #{operationLogQueryDto.requestUri}</if>
            <if test="operationLogQueryDto.params !=null and operationLogQueryDto.params !=''">and params = #{operationLogQueryDto.params}</if>
            <if test="operationLogQueryDto.exceptionInfo !=null and operationLogQueryDto.exceptionInfo!=''">and exception_info = #{operationLogQueryDto.exceptionInfo}</if>
            <if test="operationLogQueryDto.detail !=null and operationLogQueryDto.detail !=''">and detail = #{operationLogQueryDto.detail}</if>
            <if test="operationLogQueryDto.creator !=null and operationLogQueryDto.creator !=''">and creator = #{operationLogQueryDto.creator}</if>
            <if test="operationLogQueryDto.modifier !=null and operationLogQueryDto.modifier !=''">and modifier= #{operationLogQueryDto.modifier}</if>
            <if test="operationLogQueryDto.createTime !=null and operationLogQueryDto.createTime!=''">and create_time = #{operationLogQueryDto.createTime}</if>
            <if test="operationLogQueryDto.updateTime !=null and operationLogQueryDto.updateTime!=''">and update_time = #{operationLogQueryDto.updateTime}</if>
        </where>
        AND is_delete = 0 ORDER BY update_time DESC
    </select>
```

### 2. foreach遍历

```xml
<select id="selectPostIn" resultType="domain.blog.Post">
  SELECT *
  FROM POST P
  WHERE ID in
  <foreach item="item" index="index" collection="list"
      open="(" separator="," close=")">
        #{item}
  </foreach>
</select>
```

*foreach* 元素的功能非常强大，它允许你指定一个集合，声明可以在元素体内使用的集合项（item）和索引（index）变量。它也允许你指定开头与结尾的字符串以及在迭代结果之间放置分隔符。这个元素是很智能的，因此它不会偶然地附加多余的分隔符。

自己写的

```xml
<select id="queryRobotKnowledgeByIdList" resultType="com.shuidihuzhu.cs.robot.po.RobotKnowledgePo">
        SELECT <include refid="Base_Column_List"/> FROM cs_robot_knowledge
        WHERE id IN 
        <foreach collection="list" item="id" open="(" separator="," close=")">
            #{id}
        </foreach>
        AND is_delete = 0
    </select>
```

[Mybatis动态Sql](http://www.mybatis.org/mybatis-3/zh/dynamic-sql.html)

## 采坑篇

#### 1.分页查询没效果

1. 当Dao层要使用QueryDTO这种搜索对象时候（二者选其一就可）
   - 要么是在dao层的参数里面使用   queryByPage(**@Param** QueryDTO queryDTO), 需要@Param修饰
   - 要么在xml里面的 selectByPage里面采用    paramType = "com.*.QueryDTO" 明确标识出来，否则不能生效