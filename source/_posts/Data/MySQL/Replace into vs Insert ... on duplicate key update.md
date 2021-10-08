---
title: Replace into vs Insert ... on duplicate key update
categories: MySQL
tags:
  - MySQL
date: 2021-08-26 11:08:32
---
## Replace into vs Insert ... on duplicate key update

### Replace into

`REPLACE INTO ...`每次插入的时候如果唯一索引对应的数据已经存在，会删除原数据，然后重新插入新的数据，这也就导致id会增大(每次简单插入语句都会直接增大自增id)，但实际预期可能是更新那条数据。

REPLACE INTO 实际最终一定实现的是插入效果， 先删除在插入，如果插入语句中存在没有指定的字段（使其复制默认值），容易造成和预期效果不符合，比如字段create_time。 默认赋值当前时间。

当replace into未指定create_time时，如果遇见唯一索引冲突，则会先删除这条记录，然后按照INSERT INTO的效果重新插入。 此时create_Time会被重新赋值成当前时间。 因此造成了预期效果不一致，这种在主从复制的数据库采用Statement格式的binlog时候尤其明显。

###  Insert ... on duplicate key update

```
INSERT ... ON DUPLICATE KEY UPDATE`在冲突时并不会直接删除掉当前行，而是会在当前行进行更新，但是为什么这里自增地也会增大， 这里就涉及到MySQL的一个参数`innodb_autoinc_lock_mode`, 在MySQL5.1后加入，有0，1，2三种取值范围，默认值是1，之前的版本可以看做都是0。 查看方式`select @@innodb_autoinc_lock_mode;
```

innodb_autoinc_lock_mode不同参数对数据的影响

> 0，每次插入数据时都会加一个叫`AUTO_INC`的表锁，用于此事务分配AUTO_ID，在语句结束后释放，按实际是否增加数据行来分配自增di
> 1, 在`insert into select ...`等复杂语句时，加入`AUTO_INC`表锁，按照实际插入行数来分配自增id， 在INSERT_INTO VALUES等简单插入语句（知道插入行数时），会直接分配对应的AUTO_ID数量。
> 2， 任何时候都不会加上`AUTO_INC`表锁，存在安全问题。当binlog格式设置为Statement模式的时候，从库同步的时候，执行结果可能跟主库不一致，问题很大。

```mysql
truncate table t1;
insert into t1 values(NULL, 100, "test1"),(NULL, 101, "test2"),(NULL, 102, "test2"),(NULL, 103, "test2"),(NULL, 104, "test2"),(NULL, 105, "test2");

-- 此时数据表下一个自增id是7

delete from t1 where id in (2,3,4);

-- 此时数据表只剩1，5，6了，自增id还是7

insert into t1 values(2, 106, "test1"),(NULL, 107, "test2"),(3, 108, "test2");
```

**分析：**

1. 在模式1情况下，上面的例子执行完之后表的下一个自增id是10, 可以结合Mixed-mode inserts去理解
2. 模式0的话就是不管什么情况都是加上表锁，等语句执行完成的时候在释放，如果真的添加了记录，将auto_increment加1，所以这里的自增id应该就是8

**注意：**

1. 在`innodb_autoinc_lock_mode=1`时，类似`INSERT INTO t1 (c1,c2) VALUES (1,'a'), (NULL,'b'), (5,'c'), (NULL,'d')`的这种`Mixed-mode inserts`插入语句， MySQL分析语句时，会按照尽可能多的情况去分配auto_incrementid， 即使这个分配的id可能并不被使用。`INSERT ... ON DUPLICATE KEY UPDATE ...`显然也是这样的情况。
2. 在没有持有AUTO_INC表锁时， 一旦插入语句执行，即使最后事务没有被提交（回滚），AUTO_ID也会直接加1，在持有AUTO_INC时(复杂语句）, 在语句结束后就释放AUTO_INC表锁，即使事务没有提交（与行锁不同）。 这里好好理解一下模式的机制就能明白。
3. 在复杂语句时，如果插入的自增id，小于`AUTO_INCREMENT`, `AUTO_INCREMENT`不会加1, 如果是简单语句，则会直接加1
4. 在简单语句中，如果指定了自增id,且比AUTO_INCREMENT大，则AUTO_INCREMENT=当前赋值id+1.

### 两者的优劣对比

1. `replace into`简单易读，可读性强， `INSERT ... ON DUPLICATE KEY UPDATE ...`可读性较差
2. `replace into`和`INSERT ... ON DUPLICATE KEY UPDATE ...` 在5.1之后都会使自增ID加1
3. `replace into`会改变索引结构，因此效率较差，在需要较高性能时不推荐使用， `INSERT ... ON DUPLICATE KEY UPDATE ...` 会在原有基础上进行更新，性能相对replace要更好一些
4. `replace into`在未指定所有字段时，容易造成结果不符合预期。 (default: `create_time`, 自增id）

因此在使用上，优先推荐使用`INSERT ... ON DUPLICATE KEY UPDATE ...`，并在使用了自增主键的情况下，将自增主键的数据类型设置为bigint

https://www.cnblogs.com/JiangLe/p/6362770.html

### 疑惑

1. 今天再这里复现，有发现了一个奇怪的地方。 当插入的数据比自增id小时，且数据库不存在此id时，insert都不会让其自增id增加。
   replace不管存不存在，都不会增加自增id. 看来还是学艺不精

```
| t1    | CREATE TABLE `t1` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT COMMENT 'ID，自增',
  `uid` bigint(20) unsigned NOT NULL DEFAULT '0' COMMENT '用户uid',
  `name` varchar(20) NOT NULL DEFAULT '' COMMENT '用户昵称',
  PRIMARY KEY (`id`),
  UNIQUE KEY `u_idx_uid` (`uid`)
) ENGINE=InnoDB AUTO_INCREMENT=14 DEFAULT CHARSET=utf8 COMMENT='测试replace into'


mysql> select * from t1;
+----+----------+-------+
| id | uid      | name  |
+----+----------+-------+
|  1 |      100 | test1 |
|  2 |      106 | test1 |
|  3 |      108 | test2 |
|  4 |     1013 | test  |
|  5 |      104 | test2 |
|  6 |      105 | test2 |
|  7 |      107 | test  |
| 10 |     1111 | test  |
| 12 | 12111131 | test  |
| 13 |    11131 | test  |
+----+----------+-------+
```

复现问题：

```
mysql> replace into t1 values(12, 12111131, "test");
Query OK, 2 rows affected (0.00 sec)

mysql> insert into t1 values(9, 121123, "test");
Query OK, 1 row affected (0.00 sec)

mysql> select * from t1;
+----+----------+-------+
| id | uid      | name  |
+----+----------+-------+
|  1 |      100 | test1 |
|  2 |      106 | test1 |
|  3 |      108 | test2 |
|  4 |     1013 | test  |
|  5 |      104 | test2 |
|  6 |      105 | test2 |
|  7 |      107 | test  |
|  9 |   121123 | test  |
| 10 |     1111 | test  |
| 12 | 12111131 | test  |
| 13 |    11131 | test  |
+----+----------+-------+
11 rows in set (0.00 sec)
```

最后贴个结果
![8f08c50a462a4462bde5aadc9aa36c7a.png](/evernotecid:/BCD9519F-D3CB-4AFA-9D0A-5E1C3FB5B215/appyinxiangcom/26228147/ENResource/p399)

1. replace在插入数据完全相同时， 只会影响一行, 且自增主键AUTO_INCREMENT只会增加1， 在行数据不同时会影响两行，如果两行主键不同，自增主键AUTO_INCREMENT会等于替换后的主键+1.
   ![588195ce284b634068439758c8673f10.png](/evernotecid:/BCD9519F-D3CB-4AFA-9D0A-5E1C3FB5B215/appyinxiangcom/26228147/ENResource/p410)

**参考文章**

1. https://www.cnblogs.com/JiangLe/p/6362770.html
2. https://segmentfault.com/a/1190000017268633?utm_source=sf-related
3. [讲讲INSERT ON DUPLICATE KEY UPDATE 的死锁坑](