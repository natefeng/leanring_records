# Redis

## Nosql概述

```java
1.单机Mysql的年代！
```

![image-20210203214620076](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210203214620076.png)

90年代，单个数据库完全足够

这种网站的瓶颈是什么？

1.数据量太大，一个机器放不下了

2.数据的索引，一个机器的内存也放不下

3.访问量(读写混合),一个服务器承受不了

![image-20210203214756109](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210203214756109.png)

```java
2.Memcached(缓存)+Mysql+垂直拆分(读写分离)
```

网站80%的情况都是在读。

发展过程：优化数据结构和索引---》文件缓存(iO)----》Memcached

```java
3、分库分表 + 水平拆分 + Mysql集群
```

![image-20210204181413399](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204181413399.png)

```java
4丶如今最近的年代
```

如今信息量井喷式增长，各种各样的数据出现（用户定位数据，图片数据等），大数据的背景下关系型数据库（RDBMS）无法满足大量数据要求。Nosql数据库就能轻松解决这些问题。目前一个基本的互联网项目：

![image-20210204181436060](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204181436060.png)

## 为什么要用Nosql？

用户的个人信息，社交网络，地理位置。用户自己产生的数据，用户日志等等爆发式增长！这时候我们就需要使用NoSQL数据库的，Nosql可以很好的处理以上的情况！

### 什么是Nosql

NoSQL = Not Only SQL（不仅仅是SQL）

Not Only Structured Query Language

关系型数据库：列+行，同一个表下数据的结构是一样的。

非关系型数据库：数据存储没有固定的格式，并且可以进行横向扩展。

NoSQL泛指非关系型数据库，随着web2.0互联网的诞生，传统的关系型数据库很难对付web2.0时代！尤其是超大规模的高并发的社区，暴露出来很多难以克服的问题，NoSQL在当今大数据环境下发展的十分迅速，Redis是发展最快的。

### Nosql特点

1.方便扩展（数据之间没有关系，很好扩展！）

2.大数据量高性能（Redis一秒可以写8万次，读11万次，NoSQL的缓存记录级，是一种细粒度的缓存，性能会比较高！）

3.数据类型是多样型的！（不需要事先设计数据库，随取随用）

4.传统的 RDBMS 和 NoSQL

> 传统的RDBMS(关系型数据库)
>
> > 结构化组织
> >
> > SQL
> >
> > 数据和关系都存在单独的表中 row col 可以通过SQL结合特点关系查询出符合条件的数据
> >
> > 严格的一致性
> >
> > 基础的事务

> Nosql
>
> > 不仅仅是数据库 Not only Sql
> >
> > 没有固定的查询语言
> >
> > 键值对存储，列存储，文档存储，图形数据库
> >
> > 最终一致性
> >
> > CAP定理和BASE
> >
> > 高性能，高可用，高可扩

5.大数据时代的3V：主要描述问题的

海量 Velume

多样 Variety

实时 Velocaity

6.大数据时代的三高：对程序的要求

高并发

高性能

高可扩

## Redis常用命令

```bash
SINTER key1 key2 //取Set集合交集 共同好友就可以这样实现
SDIFF key1 key2 //取set差集
sadd key1 a; //给key集合加入a
无序不可重复
SUNION key1 key2 //并集
```

```bash
List是一个双向链表，LPUSH头插法，RPUSH尾插法，可以当作栈来使用，也可以当前消息队列来使用
LPUSH
RPUSH
```

```bash
Hash 相当于 key map 
hset myhash myfield1 fjh //相当于设置一个名字为myhash的map 然后map里面放入了myfield1的key value为fjh
hget myhash myfield1
hdel myhash myfield1
hgetall myhash
hsetnx myhash field4 5 //如果不存在field4这个key 则插入
hash场景应用 比如说存一个user对象 里面有name age等等 
hash更适合存储对象
```

![image-20210205170342149](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210205170342149.png)

## CAP理论

传统的Mysql比如说InnoDB是ACID

而Nosql是CAP理论

C：Consistency(强一致性)

A：Availability(可用性)

P：Partition tolerance(分区容错性)

整体网站的架构一般都要满足AP

而Nosql 如Redis,Mongodb满足CP，并且帮助整体网站满足AP

CA是创传统数据库

Base是为了解决关系数据库强一致性引起的问题而引起的可用性降低而提出的解决方案

基本可用

软状态

最终一致

![image-20210209120035275](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210209120035275.png)

## Redis配置文件

![image-20210210134713379](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210210134713379.png)

## RDF

优点：数据恢复较快，存数据，是通过写入次数来持久化的

占用空间较小，适合对数据不敏感的，安全性要求不高的，并且恢复也较快

缺点：可能会丢失最后一次的数据

## AOF

AOF持久化是通过保存被执行的命令来记录数据库状态的，所以AOF文件的大小会随着时间的流逝越来越大，所以需要重写策略

在重写的时候会fork一个子进程，子进程的数据和主进程一模一样，会来进行对AOF文件的重写，不是基于AOF文件的，而是基于当前数据库的，简化操作，但是因为是异步的，此时主进程的数据库可能还会被写入和修改文件，为了解决数据不一致的问题，增加了一个AOF缓存，这个缓存在Fork出子进程的时候使用，子进程在执行AOF文件重写的时候，主进程需要干这几件事：

执行Client命令

将写命令追加的现有的AOF文件中

将写命令追加到AOF缓冲中

![image-20210210142222377](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210210142222377.png)

最后再将AOF缓存中的内容写入到子进程的AOF文件，然后再进行覆盖

## 事务

MUTL开启一个事务，然后执行的多个命令入队到事务中，接到这些命令不会立即执行，而是放在等待执行的事务队列里面

由EXEC命令触发事务

由Watch可以进行监控一个数据或者多个，相当于乐观锁 UnWatch取消

## 主从复制

infoReplication

SlaveOf IP Port  指定为master某个主机的从机，用于容灾恢复，读写分离