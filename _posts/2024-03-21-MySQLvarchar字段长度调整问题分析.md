---
title: MySQL OnlineDDL:varchar字段长度调整问题分析
categories: [编程, MySQL ]
---

## 现象

修改表字段长度导致超时， 原表结构该字段为：
```sql
`note` varchar(45) CHARATER SET uft8mb4 COLLATE utf8mb4_general_ci DEFAULT '' COMMENT '备注'
```
修改sql：
```sql
ALTER TABLE xxx MODIFY COLUMN  note varchar(63) DEFAULT '' COMMENT '备注';
```
涉及修改字段的表数据量近3亿，执行时间超过3000s，导致Invalid Connection报错。

## 原因

更改varchar类型的字段长度，触发了Mysql Copy方式（即通过拷贝临时表的方式实现的，这期间原表只读不可写，同时也会增加一倍的存储空间）。


Mysql 8.0 支持的方式，如下图所示，一共有三种方式：Instant、In Place、Rebuilds Table【即copy，后面统一用copy说明】其中Instant最快，InPlace次之，copy最慢；另外 针对调整varchar类型长度，可以看到可以通过In place方式支持（秒级完成)

![img.png](/assets/2024/03/21/img.png)

为什么工单里调整varchar 走了copy，而没有走inplace首先看下Mysql官方说明:


![img.png](/assets/2024/03/21/img_1.png)

对于小于等于255字节以内的长度可以使用一个byte 存储。大于255个字节的长度则需要使用2个byte存储（Mysql 对varchar物理存储设计）。online ddl in-place 模式(不锁表)只支持字段的字节长度从0到255之间 或者256到更大值之间变化。如果修改字段的长度，导致字段的字节长度无法使用 1 byte表示，得使用2个byte才能表示，比如从 240 修改为 256 ，如果在默认字符集为utf8mb4的情况下，varchar(60) 修改为 varchar(64)，则DDL需要以copy模式，也即会锁表，阻塞写操作。 
注：1）字符串的字段是以字节为单位存储的，utf8 一个字符需要三个字节，utf8mb4 一个字符需要4个字节。2）varchar(N)的逻辑意义从MySQL4.1开始，varchar (N)中的N指的是该字段最多能存储多少个字符(characters)，不是字节数。

源表字段是utf8mb4，修改长度由45-&gt;63，理论上应该走Inplace，为什么没有走？工单中SQL除了修改长度，还去掉了字符集相关属性设置，而**不同字符集在255字节下存储占位计算不同**，导致Mysql 无法简单按Inplace方式执行


最终解决调整SQL如下:
```sql
ALTER TABLE xxx MODIFY COLUMN  note varchar(63) DEFAULT '' COMMENT '备注',ALGORITHM=INPLACE,LOCK=NONE;
```
- 注：在语句上加上 ALGORITHM = INPLACE, LOCK = NONE；可以保证若语句不按INPLACE方式执行则立即失败


此外，如果对表结构有较大变动，而又不想影响业务正常运行，可以使用类似copy表的方式，将源表复制出来一份，数据完全复制到新表后再进行表名替换，替换是原子的，该实践有现成的轮子如：

- pt-online-schema-change  
- OnlineSchemaChange
