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



