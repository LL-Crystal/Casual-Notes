# HiveSQL解析原理

> 之前在项目里面用过hive，一直没有系统的学习过，
现在花时间学习下，记录一下笔记
---

### Hive体系架构
![hive_architecture](image/system_architecture.png)

[官方地址：https://cwiki.apache.org/confluence/display/Hive/Design](https://cwiki.apache.org/confluence/display/Hive/Design)

Metastore和Compiler是Hive的两个核心的模块

> Metastore – The component that stores all the structure information of the various tables and partitions in the warehouse including column and column type information, the serializers and deserializers necessary to read and write data and the corresponding HDFS files where the data is stored.
---
存储仓库中各种表和分区的所有结构信息的组件，包括列和列类型信息，读取和写入数据所需的序列化程序和反序列化程序以及存储数据的相应HDFS文件。

> HiveServer2 (HS2) is a server interface that enables remote clients to execute queries against Hive and retrieve the results (a more detailed intro here). 
---
HiveServer2（HS2）是一个服务端接口，使远程客户端可以执行对Hive的查询并返回结果。

这两个模块分别对应着Hive的两个常驻进程：metastore和hiveServer2
``` 启动
hive --service metastore 1>/dev/null 2>&1
hive --service hiveserver2 1>/dev/null 2>&1
```

hiveServer2通过Driver组件接收客户端的sql查询，通过Compiler组件将SQL编译为MapReduce任务来执行，最终返回结果。

那么Hive是如何将SQL编译为MapReduce的，首先来看一下MapReduce实现基本SQL操作的原理

### MapReduce实现基本SQL操作的原理

Join的实现原理

``` join
select u.name, o.orderid from order o join user u on o.uid = u.uid;
```

在map的输出value中为不同表的数据打上tag标记，在reduce阶段根据tag判断数据来源-最基本的Join的实现

![join](image/join.png)

Group By的实现原理

``` groupby
select rank, isonline, count(*) from city group by rank, isonline;
```

将GroupBy的字段组合为map的输出key值，利用MapReduce的排序，在reduce阶段保存LastKey区分不同的key-Reduce端的非Hash聚合过程

![groupby](image/groupby.png)

Distinct的实现原理

``` distinct
select dealid, count(distinct uid) num from order group by dealid;
```
当只有一个distinct字段时，如果不考虑Map阶段的Hash GroupBy，只需要将GroupBy字段和Distinct字段组合为map输出key，利用mapreduce的排序，同时将GroupBy字段作 为reduce的key，在reduce阶段保存LastKey即可完成去重

![distinct](image/distinct.png)

当然这只是一个demo，还有更复杂的情况这里不做讨论，下面看看hive sql的解析过程

### Hive sql 的解析

hiveSQL转换成MapReduce的执行计划包括如下几个步骤

HiveSQL ->AST(抽象语法树) -> QB(查询块) ->OperatorTree（操作树）->优化后的操作树->mapreduce任务树->优化后的mapreduce任务树

参考文档：

[https://cwiki.apache.org/confluence/display/Hive/DesignDocs](https://cwiki.apache.org/confluence/display/Hive/DesignDocs)

[http://www.aboutyun.com/thread-20461-1-1.html](http://www.aboutyun.com/thread-20461-1-1.html)

[https://www.jianshu.com/p/660fd157c5eb](https://www.jianshu.com/p/660fd157c5eb)
