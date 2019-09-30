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
//Entity 实体类
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

2. 在yml或者config文件中引入依赖项

   配置文件方式

   ![UTOOLS1566530889650.png](https://i.loli.net/2019/08/23/OLtC36FJ2GVWbsg.png)

   yml方式：

   ```xml
   plugins:
       pagehelper:
         dialect: mysql
         offsetAsPageNum: true
         rowBoundsWithCount: true
         pageSizeZero: true
         reasonable: false
         params: 'pageNum=pageHelperStart;pageSize=pageHelperRows;'
         supportMethodsArguments: false
         returnPageInfo: none
   ```

3. 两种使用方式

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

> <foreach>各项介绍

**item 每一个实体类的名称**

```xml
 <foreach collection="list" item="po" open="(" separator="," close=")">
            #{po.id}
 </foreach>
```

 **index 迭代的次数，也可认为是当前执行的行号，需要做判断的时候可以用，但是一般用不着**

这里有个小应用：

场景：树状菜单拖拽，如下

![UTOOLS1569468257974.png](https://img03.sogoucdn.com/app/a/100520146/585b38d0103d58fe75853185393bf348)

每次需要拖拽排序，

- 当前的做法是前端，在传参数的时候，排好序sort一起传过来。

- 另一种做法是：直接传个list过来，sort字段不用管，按传过来的顺序，批量插入/更改

  具体的做法如下

  1. 自己建了一个tree_sort的表，如下

  ![UTOOLS1569468597701.png](https://img04.sogoucdn.com/app/a/100520146/4535cb10b82c80651a61b0fe62fd85f3)

  

  2. 代码

  ```java
  List<TreeSortPo> treeSortPos = Lists.newArrayList();
          for (int i = 0; i < 10; i++) {
              TreeSortPo sortPo = new TreeSortPo();
              sortPo.setName("第"+i+"个PO");
              treeSortPos.add(sortPo);
          }
          treeSortPoMapper.batchInsert(treeSortPos);
  ```

  mapper层

  ```xml
  <insert id="batchInsert" parameterType="com.example.demo.po.TreeSortPo" keyProperty="id" useGeneratedKeys="true">
      insert into tree_sort (`name`, sort)
      values
      <foreach collection="list" item="po" separator="," index="idx">
        (#{po.name},
        #{idx})
      </foreach>
    </insert>
  ```

  这里就可以看到 `index="idx"` ，下面插入数据的时候，直接复制`sort` 字段`#{idx}` 

  3. 效果

  ![UTOOLS1569468527312.png](https://img04.sogoucdn.com/app/a/100520146/49cc75c4ee93a5fda308c21a226df89d)

  也可将`#{idx}` 改为 `#{idx}+1`, 效果如下

  ![UTOOLS1569468571403.png](https://img03.sogoucdn.com/app/a/100520146/5546ffcd2603cedab87abb63b7af96b3)

## 3. 自动生成xml和mapper文件

1. 配置generator.xml

   ```
   <?xml version="1.0" encoding="UTF-8" ?>
   <!DOCTYPE generatorConfiguration PUBLIC
           "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
           "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd" >
   <generatorConfiguration>
   
   1    <!-- 本地数据库驱动程序jar包的全路径, 要改为自己的 -->
       <classPathEntry location="C:\Users\\Administrator\\.m2\repository\\mysql\\mysql-connector-java\\8.0.12\\mysql-connector-java-8.0.12.jar"/>
       <context id="context" targetRuntime="MyBatis3">
           <commentGenerator>
           		<!--是否关闭默认的注释   true / false -->
               <property name="suppressAllComments" value="false"/>
               <property name="suppressDate" value="true"/>
           </commentGenerator>
   
   2        <!-- 数据库的相关配置  要改为自己的-->
           <jdbcConnection
                   driverClass="com.mysql.jdbc.Driver"
                   connectionURL="jdbc:mysql://localhost:3306/mybatis"
                   userId="root"
                   password="jim777"/>
   
           <javaTypeResolver>
               <property name="forceBigDecimals" value="false"/>
           </javaTypeResolver>
   
   3        <!-- 实体类生成的位置 要改-->
           <javaModelGenerator
                   targetPackage="com.homejim.mybatis.entity"
                   targetProject=".\src\main\java">
               <property name="enableSubPackages" value="false"/>
               <property name="trimStrings" value="true"/>
           </javaModelGenerator>
   
   4        <!-- *Mapper.xml 文件的位置  sqlMapGenerator-->
           <sqlMapGenerator
                   targetPackage="mybatis/mapper"
                   targetProject=".\src\main\resources">
               <property name="enableSubPackages" value="false"/>
           </sqlMapGenerator>
   
   5        <!-- Mapper 接口文件的位置 -->
           <javaClientGenerator type="XMLMAPPER"
                                targetPackage="com.homejim.mybatis.mapper"
                                targetProject=".\src\main\java">
               <property name="enableSubPackages" value="false"/>
           </javaClientGenerator>
   
           <!-- 相关表的配置 -->
   
   6       <table tableName="blog" />
       </context>
   </generatorConfiguration>
   ```

   需要改一些内容：

   - 本地数据库驱动程序jar包的全路径（必须要改）。
   - 数据库的相关配置（必须要改）
   - 相关表的配置（必须要改）
   - 实体类生成存放的位置。
   - MapperXML 生成文件存放的位置。
   - Mapper 接口存放的位置。

   如果不知道怎么改， 请看后面的配置详解。  图中标注的都要改为自己相应的

   > 官方英文文档   http://www.mybatis.org/generator/configreference/xmlconfig.html
   >
   > 中文版本   http://mbg.cndocs.ml/index.html

## 4. Mybatis别名设计

### 1. 在mybatis_config.xml中Mybatis别名设置

```java

<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
"http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
        <!-- 对象命名空间 -->
    <typeAliases>
         <typeAlias alias="Staff" type="com.tongdatech.Staff"/>
    </typeAliases>
   <span style="white-space:pre">	</span> <!-- 映射map -->
    <mappers>
    </mappers>
```

### 2. 还可以在spring-mybatis.xml中配置：

```java

<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
		<property name="dataSource" ref="dataSource" />
		<property name="databaseIdProvider" ref="databaseIdProvider" />
		<property name="mapperLocations" value="classpath*:/mapper/*/oracle/*.xml" />
		<!-- 给映射的类配置别名 -->
 		<!-- 默认的别名是model类的首字母小写 -->
	 	<!-- 如：StaffDeploy实体类。别名为：staffDeploy -->
		<property name="typeAliasesPackage" value="com.tongdatech.demo.domain;com.tongdatech.sys.domain;com.tongdatech.hr.domain" />
		<property name="plugins">
			<list>
				<ref bean="myBatisPaginator" />
				<ref bean="nullSqlInterceptor" />
			</list>
		</property>
		<property name="configuration">
			<bean class="org.apache.ibatis.session.Configuration">
				<property name="objectFactory">
					<bean class="com.tongdatech.sys.component.mybatis.PageableObjectFactory" />
				</property>
			</bean>
		</property>
	</bean>
</bean>
```



## 5.批量处理

### 批量插入

普通Sql

```sql
INSERT INTO tb_test(ID,NAME) VALUES(1,'zhangsan');
INSERT INTO tb_test(ID,NAME) VALUES(2,'lisi');
INSERT INTO tb_test(ID,NAME) VALUES(3,'wangwu');
```

批量插入

方式一：先将数据建一张表，然后插入表，要比普通sql插入快一倍

```sql
INSERT INTO MyTable(ID,NAME)
 SELECT 4,'000'
 UNION ALL
 SELECT 5,'001'
 UNION ALL
 SELECT 6,'002' ;
```

方式二

```sql
INSERT INTO tb_test(ID,NAME) VALUES(7,'zhangsan'),(8,'lisi'),(9,'wangwu');
```

这里，我们mybatis里面用方式二

```xml
<insert id = "" >
	insert into tb_test (id, name)
	values
	<foreach collection="list" item="po" separator=",">
      <trim prefix="(" suffix=")" suffixOverrides=",">
        <if test="po.id!=null">#{po.id},</if>
        <if test="po.name!=null">#{po.name},</if>
      </trim>
    </foreach>
</insert>
```

---

### 批量更新

普通sql

```sql
UPDATE course
    SET name = CASE id 
        WHEN 1 THEN 'name1'
        WHEN 2 THEN 'name2'
        WHEN 3 THEN 'name3'
    END, 
    title = CASE id 
        WHEN 1 THEN 'New Title 1'
        WHEN 2 THEN 'New Title 2'
        WHEN 3 THEN 'New Title 3'
    END
WHERE id IN (1,2,3)
```

这条sql的意思是，如果id为1，则name的值为name1，title的值为New Title1；依此类推。

```xml
<update id="updateBatch" parameterType="list">
            update course
            <trim prefix="set" suffixOverrides=",">
             <trim prefix="peopleId =case" suffix="end,">
                 <foreach collection="list" item="i" index="index">
                         <if test="i.peopleId!=null">
                          when id=#{i.id} then #{i.peopleId}
                         </if>
                 </foreach>
              </trim>
              <trim prefix=" roadgridid =case" suffix="end,">
                 <foreach collection="list" item="i" index="index">
                         <if test="i.roadgridid!=null">
                          when id=#{i.id} then #{i.roadgridid}
                         </if>
                 </foreach>
              </trim>
              
              <trim prefix="type =case" suffix="end," >
                 <foreach collection="list" item="i" index="index">
                         <if test="i.type!=null">
                          when id=#{i.id} then #{i.type}
                         </if>
                 </foreach>
              </trim>
       				<trim prefix="unitsid =case" suffix="end," >
                  <foreach collection="list" item="i" index="index">
                          <if test="i.unitsid!=null">
                           when id=#{i.id} then #{i.unitsid}
                          </if>
                  </foreach>
           		</trim>
            </trim>
            where
            <foreach collection="list" separator="or" item="i" index="index" >
              id=#{i.id}
          </foreach>
</update>
```

> 补个小坑，  

```xml
useGeneratedKeys="true"   自动生成主键
keyProperty="id"   如果是实体类PO，则在插入库中之后，之前的Po会set插入后的id
```

## 6.强大的resultMap

> https://www.cnblogs.com/kenhome/p/7764398.html

**resultMap是Mybatis最强大的元素，它可以将查询到的复杂数据（比如查询到几个表中数据）映射到一个结果集当中。**

**resultMap包含的元素：**

```xml
<!--column不做限制，可以为任意表的字段，而property须为type 定义的pojo属性-->
<resultMap id="唯一的标识" type="映射的pojo对象">
  <id column="表的主键字段，或者可以为查询语句中的别名字段" jdbcType="字段类型" property="映射pojo对象的主键属性" />
  <result column="表的一个字段（可以为任意表的一个字段）" jdbcType="字段类型" property="映射到pojo对象的一个属性（须为type定义的pojo对象中的一个属性）"/>
  <association property="pojo的一个对象属性" javaType="pojo关联的pojo对象">
    <id column="关联pojo对象对应表的主键字段" jdbcType="字段类型" property="关联pojo对象的主席属性"/>
    <result  column="任意表的字段" jdbcType="字段类型" property="关联pojo对象的属性"/>
  </association>
  <!-- 集合中的property须为oftype定义的pojo对象的属性-->
  <collection property="pojo的集合属性" ofType="集合中的pojo对象">
    <id column="集合中pojo对象对应的表的主键字段" jdbcType="字段类型" property="集合中pojo对象的主键属性" />
    <result column="可以为任意表的字段" jdbcType="字段类型" property="集合中的pojo对象的属性" />  
  </collection>
</resultMap>
```

**如果collection标签是使用嵌套查询，格式如下：**

```xml
 <collection column="传递给嵌套查询语句的字段参数" property="pojo对象中集合属性" ofType="集合属性中的pojo对象" select="嵌套的查询语句" > 
 </collection>
```

### 应用场景

1. 树状结构查询

   ![UTOOLS1569743667746.png](https://img01.sogoucdn.com/app/a/100520146/24442ea00ed883a0534c09ff99f46f41)

   **通过这种方式，能直接查询出树状结构**

   这里的`Collection` 标签，是将查询变为`嵌套查询` 

   ```
   ofType标签      是要返回的集合中pojo对象
   select         子查询
   column         子查询的入参
   property       pojo集合的属性
   
   这里还要注意   <resultMap   type>  中的type标签，中间放的是映射的pojo对象
   ```

2. 查询po类中带list，和其他po 的

```xml
//mapper层
<select id="getAllMenu" resultMap="BaseResultMap">
      SELECT
	m.*, r.`id` AS rid ,
	r.`name` AS rname ,
	r.`nameZh` AS rnamezh
	FROM
	menu m
	LEFT JOIN menu_role mr ON m.`id` = mr.`mid`
	LEFT JOIN role r ON mr.`rid` = r.`id`
	WHERE
	m.`enabled` = TRUE
	ORDER BY
	m.`id` DESC  
</select>
  
其中ResultMap，认真学习
  <resultMap id="BaseResultMap" type="org.sang.bean.Menu">
        <id column="id" property="id" jdbcType="INTEGER"/>
        <result column="url" property="url" jdbcType="VARCHAR"/>
        <result column="path" property="path" jdbcType="VARCHAR"/>
        <result column="component" property="component" javaType="java.lang.Object"/>
        <result column="name" property="name" jdbcType="VARCHAR"/>
        <result column="iconCls" property="iconCls" jdbcType="VARCHAR"/>
        <result column="keepAlive" property="keepAlive" jdbcType="BIT"/>
        <result column="parentId" property="parentId" jdbcType="INTEGER"/>
        <association property="meta" javaType="org.sang.bean.MenuMeta">
            <result column="keepAlive" property="keepAlive"/>
            <result column="requireAuth" property="requireAuth"/>
        </association>
        <collection property="roles" ofType="org.sang.bean.Role">
            <id column="rid" property="id"/>
            <result column="rname" property="name"/>
            <result column="rnamezh" property="nameZh"/>
        </collection>
        <collection property="children" ofType="org.sang.bean.Menu">
            <id column="id2" property="id"/>
            <result column="path2" property="path" jdbcType="VARCHAR"/>
            <result column="component2" property="component" jdbcType="VARCHAR"/>
            <result column="name2" property="name" jdbcType="VARCHAR"/>
            <result column="iconCls2" property="iconCls" jdbcType="VARCHAR"/>
            <association property="meta" javaType="org.sang.bean.MenuMeta">
                <result column="keepAlive2" property="keepAlive"/>
                <result column="requireAuth2" property="requireAuth"/>
            </association>
            <collection property="children" ofType="org.sang.bean.Menu">
                <id column="id3" property="id"/>
                <result column="name3" property="name" jdbcType="VARCHAR"/>
            </collection>
        </collection>
    </resultMap>
```

简单的理解就是，`association` 查询属性是单个po，collection查询的是`list<Po>` ,为了便于理解，再看看`Menu` 类

```java
public class Menu implements Serializable {
    private Long id;
    private String url;
    private String path;
    private Object component;
    private String name;
    private String iconCls;
    private Long parentId;
    private List<Role> roles;
    private List<Menu> children;
    private MenuMeta meta;
}
```

## 采坑篇

#### 1.分页查询没效果

1. 当Dao层要使用QueryDTO这种搜索对象时候（二者选其一就可）
   - 要么是在dao层的参数里面使用   queryByPage(**@Param** QueryDTO queryDTO), 需要@Param修饰
   - 要么在xml里面的 selectByPage里面采用    paramType = "com.*.QueryDTO" 明确标识出来，否则不能生效
   
2. Mybatis中的大于小于号

   - 第一种写法：

     ```xml
     原符号 < <= > >= & ' "
     替换符号 &lt; &lt;= &gt; &gt;= &amp; &apos; &quot;
     例如：sql如下：
     create_date_time &gt;= #{startTime} and create_date_time &lt;= #{endTime}
     ```

   - 第二种写法：

     ```xml
     大于等于
     <![CDATA[ >= ]]>
     小于等于
     <![CDATA[ <= ]]>
     例如：sql如下：
     create_date_time <![CDATA[ >= ]]> #{startTime} and create_date_time <![CDATA[ <= ]]> #{endTime}
     ```
