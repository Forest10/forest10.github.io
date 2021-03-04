---
title: SpringBoot集锦
date: 2018-07-08 12:25:19
tags: [Spring,SpringBoot]
---

一.在做与JPA集成时，出现问题如下：

Caused by: java.lang.IllegalArgumentException: Not an managed type: class com.entity.****

解决：
```
1.确保实体类中@Entity使用的是javax.persistence.Entity，@Id使用的是javax.persistence.Id。

2.SpringBoot main 启动类加入@EntityScan(value = "com.xxxx.domain")

```

二:与 mybatis 做集成时打印 log(sql和参数)
解决：
```
1. mybtias.config 里 <setting name="logImpl" value="STDOUT_LOGGING" />。

2.logback 加入    <logger name="mapper xml具体的包路径" level="debug" />


```