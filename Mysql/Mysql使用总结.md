# Mysql使用总结

> creaylei 2019.07.05 
>
> 自己在使用Mysql的时候的一些知识点记录

[![github](https://img.shields.io/badge/github-mysql-brightgreen.svg)](https://github.com/creaylei/learngit)

[TOC]

## 1.建表

1. 主键为 unsigned bigint， 无符号型可以将bigint的记录数翻一倍， int的范围的-2
   147 483 648 ～ 2 147 483 647，UNSIGNED的范围类型就是0～4 294 967 295
2. 

## 2.方法论

### 1.Count(*)和Count(1)

性能一样，count(*)会返回null的记录，count(1)不会返回null记录

性能上 count(*) = count(1) > count(Primary Key) > count(Column)

> MyIsam引擎会存储行数，所以会直接返回记录数
>
> InnoDB引擎需要查询所有记录，稍微比MyIsam慢一些

### 2.永远不要在Mysql中使用utf-8

> Mysql的UTF-8不是通常意义上的UTF-8，“utf8”只支持每个字符最多三个字节，而真正的UTF-8每个字符最多四个字节。

简单概括：

- MySQL的"utf8mb4"是真正意义上的utf8
- Mysql的"utf8"是一种"专属的编码",它你能够编码的Unicode字符并不多

#### 什么是编码？什么是utf8？

​	计算机是用0和1来存储文本。 比如"C"被存为"01000011"，那么计算机在显示这个字符时候要经过两个步骤：

1. 计算机读取"01000011"，得到数字67，因为67被编码为"01000011"
2. 然后在Unicode字符集中查找67，找到对应字符"C"

几乎所有的网络应用都使用Unicode字符集，它包含了上百万个字符，最简单的编码是UTF-32，每个字符使用32位。这样做最简单，因为一直以来，计算机将32位视为数字，而计算机最在行的就是处理数字。但问题是，这样太浪费空间了。

UTF-8可以节省空间，在UTF-8中，字符“C”只需要8位，一些不常用的字符，比如“”需要32位。其他的字符可能使用16位或24位。一篇类似本文这样的文章，如果使用UTF-8编码，占用的空间只有UTF-32的四分之一左右。



## 3.知识点

### 1. Explain

语句前加Explain，可以分析一个语句的性能

`explain select * from User where user_name = 'zhangsan'`

对explain搜索的结果分析

![image-20190705191207426](/Users/zhangleishuidihuzhu.com/Pictures/wiznote/image-20190705191207426.png)

- type : 连接类型(the join type), 描述找到所需数据的扫描方式

  常见的扫描方式由快到慢: system> const > eq_ref > ref > range > index > ALL

  | system | 系统表，少量数据，不需要进行磁盘IO  |
  | ------ | ----------------------------------- |
  | const  | 常量链接                            |
  | eq_ref | 主键索引 或者 非空唯一索引 等值扫描 |
  | ref    | 非主键唯一索引等值扫描              |
  | range  | 范围扫描                            |
  | index  | 索引树扫描                          |
  | ALL    | 全表扫描                            |

#### 1. system

- *explain select \* from mysql.time_zone;*   从系统库mysql的系统表time_zone里查询，扫描类型为system, 这些数据已经到内存里，不需要进行磁盘IO, 快
- *explain select \* from (select \* from user where id=1) tmp;*     内层嵌套const返回一个临时表， 这个表在内存中， 外层在这个临时表中查找， 扫描类型也是system，不需IO

#### 2. const

- explain select * from user where id = 1  ;  

  扫描条件为： 1. 命中主键(PK)或者唯一索引(Unique)  2. 被连接部分是一个常量值， 这类扫描效率极高，返回数据量少，速度非常快

#### 3. eq_ref

- explain select * from user, user_ex where user.id = user_ex.id 

  扫描条件为： 对于前表的每一行， 后表只有一行对应

  即： 1. join查询， 2.命中主键或者非空唯一索引  3. 等值连接 

  这里 id为主键，join查询为eq_ref， 也非常快

#### 4. ref

- 上例eq_ref案例中的主键索引，改为普通非唯一(non unique)索引；对于前表的每一行(row)，后表可能有多于一行的数据被扫描
- *explain select \* from user,user_ex where user.id=user_ex.id;*
- ref扫描，可能出现在join里，也可能出现在单表普通索引里，每一次匹配可能有多行数据返回，虽然它比eq_ref要慢，但它仍然是一个很快的join类型。

#### 5. range

- range扫描比较好理解，他是索引上的范围查询，他会在索引上扫描特定范围的值
- 像上例中的**between**，**in**，**>**都是典型的范围(range)查询
- 画外音：必须是索引，否则不能批量跳过

#### 6. index

- index类型， 需要扫描**索引上的全部数据**
- 它仅比全表扫描快一点

#### 7. ALL

- *explain select \* from user,user_ex where user.id=user_ex.id;*
- 如果id上不建索引，对于前表的每一行(row)，后表都要被权标扫描

注：今天的分析中，这个相同的join语句出现了三次

1. id为主键的时候，扫描类型为eq_ref
2. id为非唯一索引，扫描类型为ref
3. id没有索引的时候，扫描类型为ALL

#### 8. 总结

（1）explain结果中的**type字段**，<u>表示（广义）连接类型</u>，它描述了找到所需数据使用的扫描方式；

（2）常见的扫描类型有：

system>const>eq_ref>ref>range>index>ALL

其扫描速度由快到慢；

（3）各类扫描类型的要点是：

- **system**最快：不进行磁盘IO
- **const**：PK或者unique上的等值查询
- **eq_ref**：PK或者unique上的join查询，等值匹配，对于前表的每一行(row)，后表只有一行命中
- **ref**：非唯一索引，等值匹配，可能有多行命中
- **range**：索引上的范围扫描，例如：between/in/>
- **index**：索引上的全集扫描，例如：InnoDB的count
- **ALL**最慢：全表扫描(full table scan)

（4）建立正确的索引(index)，非常重要；

（5）使用explain了解并优化执行计划，非常重要；

### 2. Group_concat

使用group_concat必须配合group by分组， 因为是一对多的场景

group_concat([DISTINCT]要连接的字段[order by ASC/DESC排序字段][Separator'分隔符'])

```sql
select group_concat(name) from table t where () group_by id
```



### 3.存储过程

一组为了完成特定功能的SQL 语句集，它存储在数据库中，一次编译后永久有效，用户通过指定存储过程的名字并给出参数（如果该存储过程带有参数）来执行它。存储过程是数据库中的一个重要对象。速度快

#### 创建

```sql
CREATE DEFINER=`root`@`localhost` PROCEDURE `addUser`(IN username varchar(20), IN sex tinyint(1), IN age int(20))
BEGIN
	insert into t_user set user_name=username,sex=sex,age=age;
END
```

definer:指创建者，不写为默认

IN 代表输入参数， OUT 代表输出  INOUT代表输入输出

varchar后面的(20)必须写，不写会报错

BEGIN-END中间是存储过程执行的sql

#### 使用

```sql
call addUser('ZHANG1',0,20)
```

直接调用所传的参数就可以

#### 利弊

**利**：可以减少代码逻辑，提高运行速度

**弊：**

1. 本身的运算性能会更差，远远低于外部java程序，之所以用存储过程，常常是因为某个业务逻辑过于复杂，难以用sql描述
2. 存储过程需要编译才能使用，修改算法时候，要dba的配合，造成安全性威胁
3. 不好管理
4. 能不用则不用

## 4.三范式

第一范式：1NF是对属性的原子性约束，要求属性具有原子性，不可再分解；

第二范式：2NF是对记录的惟一性约束，要求记录有惟一标识，即实体的惟一性；

第三范式：3NF是对字段冗余性的约束，即任何字段不能由其他字段派生出来，它要求字段没有冗余。

没有冗余的数据库设计可以做到。但是，没有冗余的数据库未必是最好的数据库，有时为了提高运行效率，就必须降低范式标准，适当保留冗余数据。具体做法是：在概念数据模型设计时遵守第三范式，降低范式标准的工作放到物理数据模型设计时考虑。降低范式就是增加字段，允许冗余

## 5.索引失效

1. 对于创建的多列索引（复合索引），不是使用的第一部分就不会使用索引

   ```sql
   alter table student add index my_index(name, age)   // name左边的列， age 右边的列                                                              
   
   select * from student where name = 'aaa'     // 会用到索引
   
   select * from student where age = 18          //  不会使用索引
   ```

2. 对于使用 like 查询， 查询如果是 ‘%aaa’ 不会使用索引，而 ‘aaa%’ 会使用到索引。

   `select * from student where name like 'aaa%'` // 会用到索引

   ```sql
   select * from student where name like '%aaa'        或者   '_aaa'   //  不会使用索引
   ```

3. 如果条件中有 or， 有条件没有使用索引，即使其中有条件带索引也不会使用，换言之， 就是要求使用的所有字段，都必须单独使用时能使用索引。

4. 如果列类型是字符串，那么一定要在条件中使用引号引用起来，否则不使用索引。

5. 如果mysql认为全表扫面要比使用索引快，则不使用索引。

   如：表里只有一条数据。

## 4. 错误用法

1. limit

   limit的



