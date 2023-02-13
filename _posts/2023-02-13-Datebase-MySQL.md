---
layout: post
title: "MySQL数据库"
subtitle:   "事物隔离机制"
author:     "Walklown"
date: 2023-02-13 10:45:13 -0400
background: 'img/posts/06.jpg'
tags:
    - MySQL
    - MVCC
    - 事物
---

### 1、ACID

&ensp;&ensp;ACID，是指数据库管理系统（DBMS）在写入或更新资料的过程中，为保证交易（transaction）是正确可靠的，所必须具备的四个特性：  
* 原子性（atomicity，或称不可分割性）：在同一项业务处理过程中，事务保证了对多个数据的修改，要么同时成功，要么同时被撤销。
* 一致性（consistency）：任何数据库事务修改数据必须满足定义好的规则（正确性）
* 隔离性（isolation，又称独立性）：定义了数据库系统中一个事务中操作的结果在何时以何种方式对其他并发事务操作可见。
* 持久性（durability）：定义了资料库系统中保证已提交的资料库交易（transactions）将永久存在。

&ensp;&ensp;在数据库系统中，一个事务是指：由一系列数据库操作组成的一个完整的逻辑过程。例如银行转帐，从原账户扣除金额，以及向目标账户添加金额，这两个数据库操作的总和，构成一个完整的逻辑过程，不可拆分。这个过程被称为一个事务，具有ACID特性。ACID的概念在ISO/IEC 10026-1:1992文件的第四段内有所说明。

-- 摘自维基百科

### 2、读现象

[ANSI/ISO SQL](https://zh.wikipedia.org/wiki/SQL) 92标准描述了三种不同的一个事务读取另外一个事务可能修改的数据的“读现象”。（注意，这里是各类数据库系统通用定义）

#### 脏读

&ensp;&ensp;当一个事务允许读取另外一个事务修改但未提交的数据时，就可能发生脏读（dirty reads）。

&ensp;&ensp;脏读和不可重复读类似，不同点在于事务2不需要提交就能造成语句1两次执行的结果不同。在未提交读隔离级别唯一禁止的是更新混乱，即早期的更新可能出现在后来更新之前的结果集中。

#### 不可重复读

&ensp;&ensp;在一次事务中，当一行数据获取两遍得到不同的结果表示发生了不可重复读（non-repeatable reads）.

&ensp;&ensp;在基于锁的并发控制中“不可重复读”现象发生在当执行SELECT操作时没有获得读锁或者SELECT操作执行完后马上释放了读锁； 多版本并发控制中当没有要求一个提交冲突(commit conflict)的事务回滚也会发生“不可重复读”现象。

#### 幻影读

&ensp;&ensp;在事务执行过程中，当两个完全相同的查询语句执行得到不同的结果集。这种现象称为“幻影读（phantom read）”

&ensp;&ensp;当事务没有获取范围锁的情况下执行SELECT ... WHERE操作可能会发生“幻影读”。

“&ensp;&ensp;幻影读”是不可重复读的一种特殊场景：当事务1两次执行SELECT ... WHERE检索一定范围内数据的操作中间，事务2在这个表中创建了(如INSERT)了一行新数据，这条新数据正好满足事务1的“WHERE”子句。



### 3、事物隔离级别（isolation level）

&ensp;&ensp;ANSI/ISO SQL定义的标准隔离级别（注意，这里是各类数据库系统通用定义）

#### 可串行化

&ensp;&ensp;可串行化（SERIALIZABLE）是最高的隔离级别。

&ensp;&ensp;在基于锁机制并发控制的DBMS上，可串行化要求在选定对象上的读锁和写锁直到事务结束后才能释放。在SELECT的查询中使用一个“WHERE”子句来描述一个范围时应该获得一个“范围锁”（range-locks）。这种机制可以避免“幻影读”现象。

&ensp;&ensp;当采用不基于锁的并发控制时不用获取锁。但当系统探测到几个并发事务有“写冲突”的时候，只有其中一个是允许提交的。这种机制的详细描述见“快照隔离”

#### 可重复读

&ensp;&ensp;在可重复读（REPEATABLE READS）隔离级别中，基于锁机制并发控制的DBMS需要对选定对象的读锁（read locks）和写锁（write locks）一直保持到事务结束，但不要求“范围锁”，因此可能会发生“幻影读”。

#### 读已提交

&ensp;&ensp;在读已提交（READ COMMITTED）级别中，基于锁机制并发控制的DBMS需要对选定对象的写锁一直保持到事务结束，但是读锁在SELECT操作完成后马上释放（因此“不可重复读”现象可能会发生，见下面描述）。和前一种隔离级别一样，也不要求“范围锁”。

#### 未提交读

&ensp;&ensp;未提交读（READ UNCOMMITTED）是最低的隔离级别。允许“脏读”（dirty reads），事务可以看到其他事务“尚未提交”的修改。

*通过比低一级的隔离级别要求更多的限制，高一级的级别提供更强的隔离性。标准允许事务运行在更强的事务隔离级别上。(如在可重复读隔离级别上执行提交读的事务是没有问题的)*

**在国际统一标准下**隔离级别与读现象的关系

| 隔离级别 |   脏读   | 不可重复读 | 幻读 |
| ------- | ------- | --------- | --- |
| 读未提交 | 可能发生  | 可能发生   | 可能发生   |
| 读已提交 |    否    | 可能发生   | 可能发生   |
| 可重复读 |    否    |    否     | 可能发生   |
| 串行化   |    否    |    否     | 否   |


### 4、MySQL InnoDB下通过MVCC实现的隔离级别

[MySQL5.7](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_dirty_read)

> **By default, InnoDB operates in REPEATABLE READ transaction isolation level. In this case, InnoDB uses next-key locks for searches and index scans, which prevents phantom rows (see Section 14.7.4, “Phantom Rows”).**

即：

1）MySQL的InnDB引擎中，通过MVCC机制和Gap、<font color=red>Insert Intention</font>、Record、Next-Key四种锁机制，来实现了四种隔离级别，与标准并不完全一致，更注重效率的提升。

2）MySQL的InnDB引擎中，通过Next-Key锁，使得RR隔离级别下并不会出现幻影读。

#### MySQL InnoDB 事物隔离机制

##### [InnoDB的锁](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)

Record锁：锁定记录A；  
Gap锁（表级）：锁定一个区间(A, B)，间隙可能跨越单个索引值、多个索引值，甚至是空的。
* 间隙锁是纯粹抑制性的（purely inhibitive），这意味着它们的唯一目的是防止其他事务插入间隙。 间隙锁可以共存。 一个事务获取的间隙锁不会阻止另一个事务在同一间隙上获取间隙锁。 共享和排他间隙锁之间没有区别。 它们彼此不冲突，并且它们执行相同的功能。
    * 仅与Insert Intention锁互斥，且Insert Intention Locks之间也不互斥

Next-Key锁（表级）：Record锁 + Gap锁，锁定区间(A, B]，是索引记录上的记录锁和索引记录之前的间隙上的间隙锁的组合。

##### 可重复读：

* 命中聚簇索引
    * 等值查询
        * 如果存在，加`Record`锁；
        * 如果不存在，在上一个值和下一个值之间加`Gap`锁
    * 范围查询
        * 如果存在，存在的值加`Record`锁，最后一个值后加`Gap`锁（官方文档描述为不包含下一个值的Next-Key锁，不清楚与Gap锁有什么区别）；
        * 如果不存在，在上一个值和下一个值之间加`Gap`锁；
* 命中唯一辅助索引
    * 等值查询
        * 如果存在，加`Record`锁，同时对聚簇索引加`Record`锁；
        * 如果不存在，在上一个值和下一个值之间加`Gap`锁；
    * 范围查询
        * 如果存在，存在的值加`Next-Key`锁，最后一个值后加`Gap`锁，同时对聚簇索引加`Record`锁；
        * 如果不存在，在上一个值和下一个值之间加`Gap`锁；
* 命中非唯一辅助索引
    * 等值查询
        * 如果存在，会加`Next-Key`锁，最后一个值后加`Gap`锁，同时对聚簇索引加`Record`锁；
        * 如果不存在，在上一个值和下一个值之间加`Gap`锁；
    * 范围查询
        * 如果存在，会加`Next-Key`锁，最后一个值后加`Gap`锁，同时对聚簇索引加`Record`锁；
        * 如果不存在，在上一个值和下一个值之间加`Gap`锁；
* 未命中索引
    * 加Gap锁，对匹配的行的聚簇索引加`Record`锁（未指定的话，会默认建隐藏的聚簇索引）；

##### 读已提交：

> 间隙锁定在搜索和索引扫描中被禁用，并且仅用于外键约束检查和重复键检查。

&ensp;&ensp;对于UPDATE语句，如果一行已经被锁定，则InnoDB 执行“半一致性”读取，将最新提交的版本返回给 MySQL，以便 MySQL 可以确定该行是否 匹配WHERE. UPDATE如果行匹配（必须更新），MySQL 再次读取该行，这一次InnoDB要么锁定它，要么等待锁定它。

### 5、总结

&ensp;&ensp;先了解数据库系统规范，然后再学习MySQL对于规范的实现和优化，会有更好的理解。  
&ensp;&ensp;MySQL并不是完全遵循规范的数据库，而是通过MVCC和锁，对事物进行了优化，从而提供更好的性能  
&ensp;&ensp;对于事物逻辑更准确的了解，建议读一下官方文档[InnoDB锁与事物模型](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-transaction-model.html)，本文如有错误，欢迎拍砖，共同学习进步。