---
title: mysql高级
date: 2018-07-08 12:14:46
tags: [backend,Mysql]
---

一.行级锁与表级锁

```
首先创建数据库
CREATE TABLE `test` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(32) DEFAULT NULL,
  `age` int(11) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `test_name_index` (`name`),
) ENGINE=InnoDB AUTO_INCREMENT=15 DEFAULT CHARSET=utf8mb4
```
1.1 SELECT FOR Update会引起行/表级锁(4 core)

1.11 实验有索引时候 
```
SET AUTOCOMMIT =0;START TRANSACTION ;

SELECT * FROM test WHERE id=16 FOR UPDATE ;
```
此时会造成行级锁,表内其他操作不受影响.此时的 id 可以换成 name.(原表内 name 也是有索引的).其他文章吹捧的无主键就会造成表级锁是不对的.可以自己实验.
总结来讲就是只要 where 后面跟的列是带有索引的,皆会造成行级锁

```
mysql 官网
https://dev.mysql.com/doc/refman/5.7/en/innodb-locking-reads.html

```

错误结论原文出处:http://www.cnblogs.com/chenwenbiao/archive/2012/06/06/2537508.html
![屏幕快照 2018-01-10 11.54.16.png](http://image.forest10.com/common/hexo/%E9%94%99%E8%AF%AF%E7%9A%84mysql%E8%A1%A8%E9%94%81%E7%BB%93%E8%AE%BA.png)


1.12 无索引(分为真正无索引和有索引但是索引失效)
    
```
无索引
SET AUTOCOMMIT =0;START TRANSACTION ;

SELECT * FROM test WHERE age=16 FOR UPDATE ;
```

```
有索引但是使用方式造成索引失效
SET AUTOCOMMIT =0;START TRANSACTION ;

SELECT * FROM test WHERE LENGTH(name)>33  FOR UPDATE ;
```
此时都会造成表级锁






