# MYSQL技术内幕

## Mysql体系结构和存储引擎

### 定义数据库和实例

数据库：物理操作系统文件或其他形式文件类型的集合，在Mysql数据库中，数据库可以是frm,MYD,MYI,ibd结尾的文件

实例：由后台线程和一个共享内存区域组成，后台线程可以共享该内存区域，数据库实例才是真正用于操作数据库文件的。

Mysql数据库中，实例与数据库的关系通常是一对一的，即一个实例对应一个数据库，一个数据库对应一个实例，但是在集群情况下可能存在一个数据库被多个数据实例使用的情况

### Mysql体系结构

数据库是文件的集合，是一组数据集合，而数据库实例是程序，用户对数据库的任何操作，比如数据库定义，查询，新增都是通过数据库实例下进行的，应用程序只能通过数据库实例才能和数据库打交道	

数据库可以理解为由一个个二进制文件组成，要对这些文件执行诸如INSERT,UPDATE,SELECT这样的操作，不能使用简单操作文件的方式来操作这些文件，而需要使用数据库实例来完成对数据库的操作

![image-20210126163910479](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210126163910479.png)

上图可以发现-Mysql由以下几部分组成：

连接池组件

管理和服务组件

SQL接口组件

查询分析器组件

优化器组件

缓存组件

插件式存储引擎

物理文件

**存储引擎是基于表的，而不是数据库**

### Mysql存储引擎

存储引擎可以分为MYSQL官方引擎和第三方引擎，有些第三方引擎特别强大，如InnoDb,最后被Oracle收购，其应用就极其广泛，用户也可以自己实现不同的存储引擎，或者对有些存储引擎进行扩展，每个存储引擎都有各自的特点，能够根据具体的应用建立不同的存储索引表才是关键

#### InnoDB存储引擎

支持事务，主要目标在于面向在线事务处理(OLTP)的应用

OLTP ：数据库保留的只是数据信息的最新状态，只有一个状态，适用于生产的，比如超市的买卖系统，比如每天早上洗脸照镜子，看到的就是当时的状态，至于之前的则不关心

InnoDB通过使用多版本并发控制(MVCC)来获得高并发性，并且实现了SQL标准的4种隔离级别，默认为REPEATABLE级别，同时使用一种称为next-keylocking的策略来解决幻读，除此以外，还提供了插入缓冲，二次写，自适应哈希索引，预读等高可用和高性能的功能

对于表中数据的存储，Innodb采用了聚簇的方式，因此每张表都是根据主键顺序来存放，如果没有显式指定主键，则会默认生成一个6字节的ROWID，并且以此作为主键

#### MyISAM存储引擎

**不支持事务，支持全文索引**

数据库系统和文件系统很大的不同之处在于对事务的支持，然而MyISAM是不支持事务的？为什么呢？

也是可以理解的，如果表的数据是会被查询呢？此外，该引擎的缓存池只缓存索引文件，而不缓存数据文件

MYISAM存储引擎表由MYD和MYI组成，MYD存放数据文件，MYI存放索引文件

#### NDB存储引擎

是一个集群存储索引，其特点是数据全部放在内存中，因此主键查找的速度极快，并且添加NDB结点，可以线性的提高数据库性能

这个存储引擎的问题在于连接操作是在数据库层完成的，不是在存储引擎完成的，意味这复杂的连接需要巨大的网络开销，因此查询会很慢，如果解决了该问题，NDB市场非常大

#### Memory引擎

也是将表中的数据都存放到内存，如果数据库重启或者崩溃，表中的数据都消息，非常适合存储临时表，Memory是使用的Hash索引，而不是我们熟悉的B+树索引，虽然存储引擎比较快，但是只支持表锁，并发性能较差并且不支持TEXT和BLOB列类型，MYSQL数据库使用Memory引擎来作为临时表来存放查询的中间结果集，但是如果中间结果集大于存储引擎表的容量设置，则会将该数据转换为MyISAM存储引擎表而存放到磁盘

#### Archive存储引擎

只支持INSETT和SELECT操作，非常适合存放归档数据，例如日志信息，使用行锁来实现高并发，使用zib算法进行压缩后存储，压缩比一般可达1：10，但是本身不是事务安全的存储引擎

#### Federated存储引擎

该存储引擎表并不存放数据，它只是指向一台远程Mysql数据库服务器上的某个表

#### Maria存储引擎

新开发的引擎，目标是取代MyISAM存储引擎，从而成为默认的MyISAM存储引擎，特点是支持缓存数据和索引文件，应用了行锁设计，提供了MVCC功能，支持事务和非事务安全的选项，以及更好的BLOB字符类型的处理性能

为什么Mysql不支持全文索引？ 不！mysql支持，MyISAM,Innodb都支持全文索引

![image-20210126182453383](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210126182453383.png)

### 连接Mysql

连接Mysql操作是一个连接进程和数据库实例进行通信，从程序的设计角度来讲，本质上是两个进程之间进行通信，通信的方法有TCP/IP套接字，管道等

### 安装位置

![image-20210128181914885](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210128181914885.png)

在Linux下，存放mysql数据库的数据默认在/var/lib/mysql中，	

### SQL执行顺序

![image-20210128211020460](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210128211020460.png)

上面是机器理解的 

select * from <left_table> join <right_table> on <join_condition>  where <where_condition> Group By <group_by_list> Having<having_condition> order By<order_by_condition> Limit<limit_number>

![image-20210128211257462](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210128211257462.png)

![image-20210128212558237](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210128212558237.png)

查询截取分析：

1，观察，至少跑1天，看看生产的慢SQL情况

2，开启慢查询日志，设置阈值，比如超过5秒认为就是慢SQL，并将它抓取出来

3，explain+慢SQL分析

4，show profile

5，运维经理 OR DBA,进行SQL数据库服务器的参数调优

![image-20210202173843597](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210202173843597.png)

查询优化，永远小表驱动大表

```java
select * from tbl_emp e where e.deptId in (select id from tbl_dept d);
```

上图相当于，先查出括号里面的然后通过 in关键字去查找，相当于括号里面的子查询为程序的外层for循环，而外面的为程序的里层循环

```java
select * from tbl_emp e where exists(select 1 from tbl_dept d where d.id = e.deptId);
```

上图相当于，先查出最外层的emp所有员工，然后去过滤掉不符合括号里面条件的

**in建议用于括号里面的表小，使用小表去驱动大表**

**exists使用于外面的表小，而括号里面的表特别大适合**

优化总结口诀：

全值匹配我最爱，最左前缀要遵守

带头大哥不能死，中间兄弟不能断

索引列上少计算，范围之后全失效

Like百分写最右，覆盖索引不写星，

不等空值还有OR，索引失效要少用，

VAR引号不可丢，SQL高级也不难

**order by 满足两种情况下，会使用index方式排序**

order by语句使用索引最左前列

使用where子句与order by子句条件组合满足索引最左前列

提高order by的方法：

1.尽量不然使用select * from 因为filesort是基于buffer对其数据进行排序的，单路排序会缓存查询所需要的所有数据，如果是*则缓存一行数据，造成数据量大，需要创建tmp表来存取多余的数据造成多次合并，性能低。单路缓存所需要的所有数据，双路只缓存主键和需要排序的字段数据，然后通过主键回表再去查询，两次IO

2.如果发现单路和双路都会发生多次IO，那么可以提高sort_buffer_size的值

### 面试题自我理解

**前言**

**涉及到的知识点／你可以了解到的点，关键字**

**索引原理，底层存储；**

**B-Tree、B+Tree**

**聚集索引，非聚集索引，联合索引，覆盖索引**

**为什么会索引失效／索引失效的原理**

什么是索引？

索引是为了加速对表中数据行的检索而创建的一种数据结构

大白话就是：索引是一种排好序便于数据查找的数据结构

为什么加索引？

索引可以把随机IO变为顺序IO

索引能极大的减少存储引擎需要扫描的数据量

索引可以帮助我们进行分组，排序等操作时避免使用临时表

show profiles； 性能查看工具

如果出现converting HEAP to MyISAM 查询结果太大，内存不够用向磁盘搬

Creatimg to tmp 创建临时表，拷贝数据到临时表，用完再删除

Copying to tmp table on disk 把内存种临时表复制到磁盘，危险！

locked



一个会话中使用Myisam引擎去使用读锁锁一个表，然后可以读该表，但是不能写该表，也不能查看和更新其他未锁定的表，因为还未释放

另外一个会话可以去查被读锁锁定的表，并且可以修改和查看其他未被锁定的表，但是不能写被读锁锁定的表，因为会阻塞等待，直到读锁释放

一个会话中使用Myisam引擎去使用写锁锁一个表，然后可以读该表，写该表，不能查看和更新其他未锁定的表，因为还未释放当前写锁

另外一个会话可以不能查被写锁锁定的表，更不能修改，因为会阻塞等待，直到写锁释放

**读锁会阻塞写，但是不会阻塞读，而写锁则是把读和写都阻塞**

如何分析表锁定？ 可以通过table_locks_waited和table_locks_immediate状态变量来分析系统上的表锁定。

show status like 'table';

**table_locks_immediate**：该参数代表产生表级锁定的次数，可以获取锁的查询次数

**table_locks_waited**：出现表级竞争而发生等待的次数，如果过高则代表有严重的表级锁正用现象

Myisam会给每次读操作加读锁，写操作加写锁

immediate表示的就是产生锁的次数

![image-20210203161348573](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210203161348573.png)

Myisam写优先，大量的写会造成查询查不到，饥饿现象

### 行锁理论

偏向于InnoDB存储引擎，开销大，加锁慢，会出现死锁，锁粒度最小，发生锁冲突的概率最低，并发度也最高

![image-20210203162646027](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210203162646027.png)

更新丢失，事务更是操作一行，两个操作其中一个操作会被覆盖，比如两个程序员同时打开一个共享文件进行修改，覆盖的时候会覆盖掉另外一个人更改过的内容

脏读，就是读未提交，指读取到了别人还已修改未提交事务的数据

不可重复读：就是指读取了一次数据与第二次数据不一致，不符合一致性

幻读：是指事务A读到了事务B已提交新增的数据，但是读不到修改的数据，会读到新增加的数据，不符合隔离性

ACID 原子性，一致性，隔离性，持久性

一致性：是指事务开始之前和事务结束之后，数据的完整性约束没有被破坏，这是说数据库事务不能破坏数据的完整性以及业务逻辑的一致性

比如转账操作，不管事务成功还是失败，两个账户的总金额不能变

隔离性：是指多个事务并发访问时，事务之间是隔离的，一个事务不应该影响其他事务的运行

**事务之间的影响会造成：脏读，不可重复读，幻读，丢失更新，也就是上面所指**

事务的四种隔离级别：

未提交读：解决丢失更新

已提交读：解决脏读

可重复读：解决不可重复读

可序列化：解决所有

![image-20210203164433332](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210203164433332.png)

![image-20210203172804721](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210203172804721.png)

索引失效会导致行锁升级为表锁

为什么呢？ 

间隙锁：当我们使用范围条件而不是相等条件检索数据的时候，并请求共享或者排他锁的时候，InnoDB会给符合条件的已有数据记录的索引项加锁，对于健值在条件范围内但是不存在的记录叫做间隙

InnoDB也会对这个间隙加锁，这个锁机制就是所谓的间隙锁

比如一个对其大于1小于6的数据进行范围修改，另外一个对该表进行插入2号，原本数据表中只有3，4，5。但是2号在插入的时候也会被阻塞，因为'间隙'被锁了

注意：如果使用InnoDB的表使用行锁，被锁定的字段不是主键，也没有针对该字段建立索引，那么锁定的就是整张表

如何加行锁？

select * from test_innodb_lock where a = 8 for update;  后面的for update含义就是此时将 a = 8 的行锁定

![image-20210203183844602](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210203183844602.png)

show status like 'innodb_row_lock%';

![image-20210203184023659](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210203184023659.png)

### 主从复制

Mysql复制过程分为三步：

1.master将改变记录到二进制日志(binary log)中，这个记录过程叫做二进制日志事件，binary log events

2.slave将master的binary log events拷贝到它的中继日志(relay log)

3.slave重做中继日志中的事件，将改变应用到自己的数据库中，Mysql的复制是异步且串行话的

![image-20210203184737676](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210203184737676.png)

IO线程在读取到Relay log，然后SQL thread读取到数据库发生更改

### Explain

explain模拟优化器执行Sql,在5.6以及以后的版本中，除过select，其他比如insert,update,delete均可以使用explain查看执行计划

```java
信息	描述
id	查询的序号，包含一组数字，表示查询中执行select子句或操作表的顺序
**两种情况**
id相同，执行顺序从上往下
id不同，id值越大，优先级越高，越先执行
select_type	查询类型，主要用于区别普通查询，联合查询，子查询等的复杂查询
1、simple ——简单的select查询，查询中不包含子查询或者UNION
2、primary ——查询中若包含任何复杂的子部分，最外层查询被标记
3、subquery——在select或where列表中包含了子查询
4、derived——在from列表中包含的子查询被标记为derived（衍生），MySQL会递归执行这些子查询，把结果放到临时表中
5、union——如果第二个select出现在UNION之后，则被标记为UNION，如果union包含在from子句的子查询中，外层select被标记为derived
6、union result:UNION 的结果
table	输出的行所引用的表
type	显示联结类型，显示查询使用了何种类型，按照从最佳到最坏类型排序
1、system：表中仅有一行（=系统表）这是const联结类型的一个特例。
2、const：表示通过索引一次就找到，const用于比较primary key或者unique索引。因为只匹配一行数据，所以如果将主键置于where列表中，mysql能将该查询转换为一个常量
3、eq_ref:唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配。常见于唯一索引或者主键扫描
4、ref:非唯一性索引扫描，返回匹配某个单独值的所有行，本质上也是一种索引访问，它返回所有匹配某个单独值的行，可能会找多个符合条件的行，属于查找和扫描的混合体
5、range:只检索给定范围的行，使用一个索引来选择行。key列显示使用了哪个索引，一般就是where语句中出现了between,in等范围的查询。这种范围扫描索引扫描比全表扫描要好，因为它开始于索引的某一个点，而结束另一个点，不用全表扫描
6、index:index 与all区别为index类型只遍历索引树。通常比all快，因为索引文件比数据文件小很多。
7、all：遍历全表以找到匹配的行
注意:一般保证查询至少达到range级别，最好能达到ref。
    possible_keys	指出MySQL能使用哪个索引在该表中找到行
key	显示MySQL实际决定使用的键(索引)。如果没有选择索引,键是NULL。查询中如果使用覆盖索引，则该索引和查询的select字段重叠。
key_len	表示索引中使用的字节数，该列计算查询中使用的索引的长度在不损失精度的情况下，长度越短越好。如果键是NULL,则长度为NULL。该字段显示为索引字段的最大可能长度，并非实际使用长度。
ref	显示索引的哪一列被使用了，如果有可能是一个常数，哪些列或常量被用于查询索引列上的值
rows	根据表统计信息以及索引选用情况，大致估算出找到所需的记录所需要读取的行数
Extra	包含不适合在其他列中显示，但是十分重要的额外信息
1、Using filesort：说明mysql会对数据适用一个外部的索引排序。而不是按照表内的索引顺序进行读取。MySQL中无法利用索引完成排序操作称为“文件排序”
2、Using temporary:使用了临时表保存中间结果，mysql在查询结果排序时使用临时表。常见于排序order by和分组查询group by。
3、Using index:表示相应的select操作用使用覆盖索引，避免访问了表的数据行。如果同时出现using where，表名索引被用来执行索引键值的查找；如果没有同时出现using where，表名索引用来读取数据而非执行查询动作。
4、Using where :表明使用where过滤
5、using join buffer:使用了连接缓存
6、impossible where:where子句的值总是false，不能用来获取任何元组
7、select tables optimized away：在没有group by子句的情况下，基于索引优化Min、max操作或者对于MyISAM存储引擎优化count（*），不必等到执行阶段再进行计算，查询执行计划生成的阶段即完成优化。
8、distinct：优化distinct操作，在找到第一匹配的元组后即停止找同样值的动作。
```

简单来说 const是直接按照主键或者唯一索引读取，而eq_ref则是用于连表查询的情况下，按联表的主键或者唯一索引联合查询

## InnoDB存储引擎

![image-20210204120459918](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204120459918.png)

InnoDB适合于OLTP应用，OLTP为称为联机事务处理，OLAP被称为联机分析处理

字面上来看OLTP是做事务处理，OLAP是做分析处理

从数据库操作来看，OLTP是增删改，OLAP是查询，正是因为InnoDB的存在，才让Mysql数据库变得更有魅力

### InnoDB存储引擎概述

该存储引擎是第一个完整支持ACID事务的Mysql存储引擎(BDB是第一个事务的存储引擎)，其特点是行锁，支持MVCC，提供外键，提供一致性锁定读，同时被设计以来最有效利用以及使用内存和CPU，Mysql官方手册可知，InnoDb上处理插入/更新操作的速度平均为800次/秒，这些都证明了InnoDB是一个高性能，高可用，高可扩展的存储引擎

### InnoDB存储引擎的版本

InnoDB存储引擎被包含于所有的Mysql数据库的二进制发行版本中，早期随着Mysql数据库的更新而更新，从Mysql5.1版本时，Mysql允许存储引擎开发商以动态的方式加载，一种是静态编译的InnoDB版本，一种是动态编译的InnoDB版本

![image-20210204121845907](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204121845907.png)

### InnoDB体系结构

![image-20210204122011405](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204122011405.png)

后台线程主要负责刷新内存中的数据，保证缓存池中的内存缓存是最近的数据。此外将已修改的数据文件刷新到磁盘文件，同时保证数据库发生异常的情况下InnoDB能恢复到正常运行状态

#### 后台线程

InnoDB存储引擎是多线程的模型，其中后台有多个不同的线程，负责处理不同的任务

##### Master Thread

Master Thread是一个非常核心的后台线程，主要负责将缓存池的数据异步刷新到磁盘，**保证数据的一致性(因为数据修改后可能会先保存到缓冲池)，包括脏页的刷新，合并插入缓冲，UNDO页的回收等**

##### IO Thread

在InnoDB大量使用了AIO来处理写IO请求，这样可以极大提高数据库性能。而IO Thread的工作主要是负责这些IO请求的回调。Innodb1.0之前共有4个IO Thread,分别是write,read,insert buffer和logIO thread。从InnoDB1.0x版本开始，read thread和write thread分别增大到了4个

```java
show variables like 'innodb_version'; //使用该参数查看Innodb版本
show variables like 'innodb_%io_threads'; //使用该参数查看IO线程
show engine innodb status;//使用该参数来观察Innodb的所有状态
```

![image-20210204123044005](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204123044005.png)

![image-20210204123129286](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204123129286.png)

主要功能：响应IO请求的回调，因为发出IO请求需要获得结果或者其他操作

![image-20210204123354012](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204123354012.png)

可以看到读线程ID总是小于写线程ID

##### Purge(清除) Thread

undolog日志(回滚日志)

事务被提交后，该日志可能不需要，因此该线程就是回收已经使用并分配的undolog页，innodb1.1版本之前在master thread中完成，1.1开始，purge操作可以到单独的线程中执行 1.2开始，InnoDB支持多个Purge Thread，目的是为了加快undo页的回收

##### Page Cleaner Thread

该线程是在Innodb1.2之后引入的，其作用是将之前版本中脏页的刷新操作放到单独的线程来操作，而目的是为了减轻原Master Thread的工作以及对于用户查询线程的阻塞

#### 内存

##### 缓冲池

Innodb存储是基于磁盘存储的，并将其中的记录按照页的方式进行管理.因此可将其视为基于磁盘的数据库系统。由于磁盘和CPU差异较大，所以使用缓冲池技术来提高性能，简单来说就是一块内存区域，在数据库进行读取页的操作，首先将磁盘存放在缓冲池中。下一次再读取相同的页时，首先判断是否在缓冲池中，如果在直接读取，否则读取磁盘上到页。

对于数据库页的修改操作，则首先修改在缓冲池中的页，然后以一定频率写回到磁盘。这里的一定频率是基于checkPoint的机制刷新回硬盘。

综上所述，缓冲池的大小直接影响数据库的性能

![image-20210204154818947](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204154818947.png)

具体来看，缓冲池缓冲的数据有：索引页，数据页，undo页，插入缓冲，自适应哈希索引，InnoDB存储的锁信息，数据字典信息。**不能简单认为缓冲池只缓冲索引页和数据页**，他们只是占缓冲池的很大一部分

索引页和数据页都以文件的形式存在于磁盘

![image-20210204155126316](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204155126316.png)

从1.0x开始允许有多个缓冲池实例。每个页根据哈希值平均分配到不同的缓冲池实例中

![image-20210204155337153](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204155337153.png)

![image-20210204155414792](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204155414792.png)

可以根据INNODB_BUFFER_POOL_STATUS来观察缓冲的状态

##### LRU List丶Free List 和 Flush List

我们已经知道了缓冲池是一个很大的内存区域，其中放着各种类型的页，那么存储引擎是怎么样对缓冲池进行管理的呢？

通常来说，数据库的缓冲池是通过LRU(最近最少使用)算法来管理的，频繁使用的页在最前端，最少使用的页在尾端，当缓冲池不能缓冲更多的页时首先会释放LRU尾端的页。

在Innodb缓冲池中，页的大小默认为16kb，同样使用LRU算法来进行管理，不过与传统的LRU算法不一致，新加入的页不是放在前端，LRU列表还加入了midPoint，而是放在长度大于为midPoint的位置，默认配置下，该位置在LRU长度列表的5/8处

![image-20210204161058563](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204161058563.png)

默认为37，表示新读取到的页插入到LRU尾端百分之37的位置，大约3/8。InnoDB中，把midPoint之后的列表称为old列表，之前的列表称为new列表。可以简单的理解为new列表中的页都是为最为活跃的数据

如果再次访问存在于old列表中的页，则会将其放入new列表

不采用朴素的LRU算法原因是某些操作会使得LRU热点数据被刷出，影响缓冲区的效率。比如索引和数据的扫描操作，可能会扫描表中的的许多页，而这些页通常来说仅在这次查询中需要，并不是真正活跃的热点数据，如果将该页放在LRU的首部，可能会将真正的热点数据移除。而在下一次读取页的时候，InnoDB存储引擎会再次访问磁盘，造成性能下降。

（内存的80%给DB，DB的80%给热数据（在一个时间段内被频繁的访问！！））
即，一个64G的物理内存，80%给数据库，数据库的80%给热数据区，即40.96G给了热数据区。

为了解决这个问题，InnoDB引入了另外一个参数来进一步管理LRU列表，这个参数是innodb_old_blocks_time，这个参数表示页读取到mid位置后多久被加入到数据的热端

```java
set global innodb_old_blocks_time=1000； //可以设置
```

代表的意思是放在热冷数据交界处，如果过一秒后还存活，则加入到热端

![image-20210204163049877](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204163049877.png)

```java
冷热数据的监控
    Pages made young 0, not young 0
    0.00 youngs/s, 0.00 non-youngs/s
```

- youngs/s：坚持到了1s，进入了热数据区；
- non-youngs/s：没有坚持到1s，于是被刷出去了。

**non-youngs/s的值过大原因**

- 1.可能存在严重的全表扫描（频繁的被刷出来）
- 2.可能是pct设置的过小（冷数据区就很小，来一点数据就刷出去了）
- 3.可能是time设置的过大（没坚持到1万s，被刷出去了）

**youngs/s的值过大的原因**

正常不可能一直很高，因为热数据区就那么大，不可能一直往里调。

- 1.pct过大（pct过大，冷区大，热区小，time过小，就容易young进去。）
- 2.time过小

LRU列表用来管理已经读取的页，但是数据库刚启动的时候，LRU列表为空，即没有任何的页。这时页都存在于Free列表中。为什么呢？ 因为操作系统会去申请一块内存区域，然后数据库会按照默认的缓存页大小划分出来一个个缓存页用来存放从磁盘读取的页，所以需要首先从Free查看是否有还有空闲的页，因为没有空闲的页都不存在于Free的双向链表之中，如果有空闲的页则取出该页，同时将该当前页从Free列表删除加入到LRU列表，否则根据LRU算法，淘汰LRU列表末尾的页，将该内存空间分配给新的页。

![image-20210204171358489](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204171358489.png)

通过上面该命令可以看到 Buffer pool Size有327679个页，327679*16K，也就是5G的缓冲池,Free buffers代表当前空闲的页，Database pages代表LRU列表中的页。还有一个重要变量

Buffer pool hit rate ，表示缓冲池的命中率，通常在95%以上，如果低于95%，用户需要观察是否是全表扫描而引起的LRU被污染的问题

![image-20210204171938305](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204171938305.png)

![image-20210204172136668](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204172136668.png)

并且从1.0x开始支持压缩页的功能，可以将原本16k的页压缩为1kb，2kb，4kb，8kb，由于页的大小发生变化，LRU列表也发生了一些变化，对于非16KB的页是通过unzip_LRU列表进行管理的

![image-20210204172329492](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204172329492.png)

LRU len长度包括unzip。

unzip_LRU对不同压缩页大小的页进行分别管理。其次，通过算法进行内存的分配。例如从需要对缓冲池申请4KB大小的页，其过程如下：

1）首先需要去查看4KB的unzip_LRU列表，检查是否有空闲的页；

2）若有直接使用

3）否则检查8K的页

4）如果有8K的页则进行拆分为两个4K的页，使用其中一个4K的页，否则从LRU列表申请一个16K的页，将其分为1个8KB的页，2个4KB的页放入unzip_lru列表中

![image-20210204172800553](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204172800553.png)

在LRU列表上的页被修改后，称为脏页，即缓冲池的页和磁盘的页数据不一致，这时候数据库会通过checkPoint机制进行刷新回磁盘，而Flush列表中的页即为脏页列表，脏页即存在于LRU列表也存于Flush列表，互补影响

![image-20210204173005012](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204173005012.png)

当然，缓存池毕竟还是内存，如果机器宕机，内存里的数据就丢失了，mysql 为了保证数据不丢失，于是引入了 redo log，undo log，bin log等一系列日志，为的就是不丢失数据，我们看下图。

![image-20210204170531530](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204170531530.png)

##### 重做日志缓冲

重做日志(redo log)，是数据库的事务日志，通常用于回滚，也可以如下操作：

1.系统崩溃后的实例恢复

2.通过备份恢复数据文件

3.备用数据库处理

InnoDB存储引擎首先将重做日志信息写入到缓冲区，然后按照一定频率刷新到日志文件

![image-20210204174136517](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210204174136517.png)

- Master Thread每一秒将重做日志文件缓冲刷新到重做日志文件
- 每个事务提交时候会将重做日志缓冲刷新到重做日志文件
- 当重做日志缓冲池剩余空间小于1/2时，重做日志缓冲刷新到日志文件

##### 额外的内存池

InnoDB内存结构整体上看大致可以分为额外的内存池，重做日志缓冲，缓冲池。而缓冲池里面有又插入缓冲，数据页等等

在InnoDB引擎中，对内存管理方式是通过一种内存堆的方式进行的，在对一些数据结构本身的内存进行分配时候，需要额外的内存池进行申请，例如，分配了缓冲池，但是每个缓冲池中的还有一些对应的缓冲控制对象，这些对象还记录了一些诸如LRU，锁，等待等信息，这些信息需要从额外内存池申请

### CheckPoint技术

频繁将缓冲池的脏页刷新回磁盘影响性能，并且如果将内存中的脏页数据刷新到磁盘时发生宕机，那么数据就不能恢复了。为了避免数据丢失的问题，当前事务数据库系统普遍采用了Write Ahead Log策略，**即当事务提交时，先写重做日志，再修改页**。当由于发生宕机时，通过重做日志来完成数据的恢复。这也是事务中ACID持久化的要求

思考两个场景? 是不是如果缓冲池足够大，是不是就可以缓存数据库全部的数据，是不是就不需要将缓冲池的数据刷新回磁盘？因为宕机的时候完全可以通过重写日志文件来恢复数据

但是这样需要两个前提条件：

缓存池中可以缓存数据库的所有数据  

重写日志文件可以无限增大

上述两个条件很难解决，即使解决了还有一个情况需要考虑

数据库的恢复时间？ 如果一个数据库运行了几个月甚至几年呢? 如果发生了宕机，那么进行恢复文件的时间将会非常非常久

因此CheckPoint(检查点)技术的目的主要是解决以下的几个问题

- **缩短数据库的恢复时间**
- **缓冲池不够用时，将脏页刷新回磁盘**
- **重做日志不可用时，刷新脏页**

当数据库发生宕机的时候，不需要重做所有的日志，只需要在checkPoint的数据进行恢复即可，大大缩短时间

当缓存池不够用时，根据LRU算法会溢出最少使用的页，如果该页为脏页那么需要强制执行checkPoint来将数据刷新回磁盘

重做日志不可用的原因是因为重做日志文件并不是让其无线增大的。而是循环利用的，被重用的部分指的是已经不需要这些重做日志文件，发生宕机的时候不需要这些日志文件来恢复，那么就可以覆盖掉，如果重做日志的数据还需要，那么就必须执行checkPoint，将缓冲区的页至少刷新到当前重做日志的位置。

对于InnoDB而言，是通过LSN(Log Sequence Number)来标记版本的，也就是日志序列号	

![image-20210205175526155](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210205175526155.png)

在InnboDB存储引擎中，CheckPoint发生的时间，条件及脏页的选择都是十分复杂的

而CheckPoint所作的事情无外乎就是将缓冲池中的脏页刷新回磁盘。不同指出就是何时刷新，每次刷新多少页，每次从哪里取脏页，在InnoDB存储引擎有两种CheckPoint，分别为：

- Sharp CheckPoint
- Fuzzy CheckPoint

Sharp CheckPoint发生在数据库发生关闭的时候将所有的脏页刷新回磁盘，这是默认的工作方式，即innodb_fast_shutdown = 1

运行时使用Fuzzy CheckPoint

![image-20210205180452891](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210205180452891.png)

**对于Master Thread中发生的CheckPoint，差不多以每秒和每十秒的速度将缓冲池列表的脏页刷新回磁盘，是异步的**

**Flush_LRU_LIST CheckPoint** 是因为LRU列表会从尾部溢出最近最少使用的页，如果这些页中有脏页那么需要进行CheckPoint，而脏页又是来自LRU列表的，所以称之为 Flush_LRU_lIST CheckPoint,1.2以后这个检查被放在了一个单独的线程Page Cleaner线程中进行

![image-20210205182325079](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210205182325079.png)

**Async/Sync Flush CheckPoint**

指的是重做日志文件不可用的情况下(因为在写数据的时候会同时写重做日志文件，如果想要覆盖的重做日志中的数据在缓冲池中还存在页，那么会进行将缓冲池的页刷新)，需要将页刷新到磁盘，Mysql 5.6之后，也同样将该操作放入到了另外一个线程，该操作是为了重做日志的循环可使用性

![image-20210205182748189](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210205182748189.png)

**Dirty Page too much CheckPoint**

顾名思义，脏页太多也会执行CheckPoint，这个多少是可以根据参数来进行设置Innodb_max_dirty_pages_pct控制

![image-20210205183635981](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210205183635981.png)

默认值为75，表示缓冲区脏页的数据占据75%的时候，会强制进行CheckPoint

### Master Thread 工作方式

我们知道InnoDB存储引擎的主要工作都是在一个单独的后台线程Master Thread中完成的。

#### InnoDB1.0x版本之前的Master Thread

**该线程具有最高的线程优先级别，其内部有多个循环组成：主循环，后台循环，刷新循环，暂停循环。Master Thread会根据数据库的运行状态在loop,backgroup loop(后台循环),flush loop 和suspend loop中进行切换**

Loop为称之为主循环，因为大多操作都在该线程执行，其中有两大部分的操作-每秒钟的操作和每10秒的操作

伪代码：

![image-20210205195716025](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210205195716025.png)

可以看到,loop循环时通过thread.sleep来完成的，因此在负载很大的情况下会有延时，只能说大概情况下为每秒和每10秒

每秒一次的操作：

- 日志缓冲刷新到磁盘，即使这个事务还没有提交(总是)
- 合并插入缓冲(可能)
- 至多刷新100个InnoDB的缓冲池中的脏页到磁盘
- 如果没有用户活动，则切换到backGroup loop(可能)

即使某个事务没有提交，InnoDB仍然每秒将重做日志的内容刷新到磁盘

合并插入缓冲的前提是前一秒内发生IO的次数是否小于5次，如果是才进行插入缓冲

同样，刷新脏页也是根据当前缓冲池脏页的比例(buf_get_modified_ratio_pct)是否超过了innodb_max_dirty_pages_pct，默认为90%，如果超过了才将100个脏页刷新回磁盘

![image-20210205200518426](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210205200518426.png)

每10秒一次的操作：

- 刷新100个脏页到磁盘(可能)

- 合并至多5个插入缓冲(总是)

- 将日志缓冲刷新到磁盘(总是)

- 删除无用的Undo页(总是)

- 刷新100个或者10个脏页到磁盘(总是)

  

Innodb会判断过去10秒内，IO读写次数是否超过200次，如果没有超过，则刷新100个脏页到磁盘

接着会执行full purge操作,删除无用的undo页，该页是存放被修改之前的值，有时候会不需要，每次尝试回收20个该页

然后判断是否脏页比例如果有超过百分之70，则进行刷新100个脏页到磁盘，否则刷新10个

![image-20210205201145349](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210205201145349.png)

**看完了主循环我们再来看backgroup loop**

若当前没有用户活动(数据库空闲时)或者数据库关闭，就会切换到该循环。会执行以下操作：

删除无用的undo页(总是)

合并20个插入缓冲(总是)

跳回到主循环(总是)

不断刷新100个页直到符合条件(可能，跳转到flush loop中完成)

可以看到做的工作就是删除undo页和合并20个插入缓冲

undo页：保存修改之前的数据，最经典的应用就是事务回滚，需要用到修改之前的数据，如果事务提交那么可能会删除该页

插入缓冲：是指插入辅助索引不会立刻同步回磁盘，而是先存于缓冲区，后面会合并

**如果flush loop也没有事情可做，InnoDB会切换到suspend loop，将Master Thread挂起，等待事件的发生，若用户启用了InnoDB存储引擎，却没有使用任何InnoDB存储引擎的表，那么总是处于挂起状态**

#### InnoDB1.2x版本之前的Master Thread

1.0x之前版本的该线程，由于对IO读写有限制，因为每次只会最多刷新100个脏页到磁盘，在高负载的情况下会很慢，然后提出了**InnoDB_io_capactity**，用来表示磁盘IO的吞吐量，默认值为200

在合并插入时，合并插入缓冲的数量为该值的%5;

在刷新脏页时，刷新脏页的数量为该值

该参数也是可以进行调整的，并且innodb_max_dirty_pages_pct该值从90%到75%

![image-20210205203243544](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210205203243544.png)

还加入了一个自适应的刷新，通过重做日志的速度来决定最合适的刷新脏页数量

![image-20210205203516498](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210205203516498.png)

压力大的时候并不总是等待1秒，如果差值越大，可能数据库负载压力越大

#### InnoDB1.2x版本的Master Thread

将10秒操作和每秒操作分开来，并且对于脏页的刷新从Master Thread分离到了另外一个线程Page cleaner Thread

### InnoDB关键特性

特性包括

- 插入缓冲(insert buffer)

- 两次写

- 自适应哈希索引

- 异步IO

- 刷新领接页

  

#### 插入缓冲

**InsertBuffer**和数据页一样，是物理页的一部分。缓冲池里面有insertBuffer的信息

在InnoDB中，主键是行唯一的标识符，通过应用程序中行记录的插入顺序是按照主键递增的顺序进行插入的

因此，插入聚集索引(Primary key)，一般是顺序的，并不是所有的主键都是顺序的，若主键是UUID这样的，那么插入和辅助索引一样，同样是随机的

![image-20210205205706520](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210205205706520.png)

聚集索引，也叫聚簇索引

> 数据行的物理存储和列值的逻辑顺序相同，一个表中只能拥有一个聚集索引

![image-20210206114401294](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210206114401294.png)

但是不可能一张表上只有一个聚集索引，更多情况下还是非聚集的辅助索引

InnoDB存储引擎存在Insert Buffer，对于非聚集索引的插入操作或者更新操作，不是每次都插入到索引页，而是先判断非聚集索引页是否在缓冲池中，如果在直接插入，若不在，则先放入到一个Insert Buffer对象上。这时候对外来说已经插入到叶子节点，实际并没有，而是以一定的频率将Insert Buffer和辅助索引页子节点的合并操作

这是通常将多个Insert Buffer对象插入到一个索引页，提交效率(因为在一个索引页中)，可以想象一下多个Insert Buffer对象一起插入到一个索引页中。比如说插入4个用户的数据，金额都为1000，1，999，2。并且金额字段是非聚聚索引并且是非唯一的。

那么如果没有插入缓冲的话，那么首先要插1000，首先需要读取1000所在的索引页。再读取1插入以此类推，可能它们不处于同一个索引页中，但是如果使用插入缓冲，那么999和1000肯定在一个索引页，1和2肯定在一个索引页，减少了读取索引页的次数。

如果存在插入多个非聚集索引，那么会造成插入复杂，不停的离散读取索引所在的索引页，出现了合并索引。

Insert Buffer的使用需要满足两个条件

- **索引是辅助索引**

- **索引不是唯一的**

  

如果不是辅助索引那么就没有了意义，因为本身聚集索引就是连续的。进行合并插入没有意义。为什么没有意义？

Why？ 因为本来在索引文件中就是连续的，不需要离散读取索引页，所以直接插入就可以

还有为什么必须不是唯一的？因为数据库并不会去判断插入记录的唯一性，因为如果再去查找又是离散读取的情况了，这样就没有意义了。因为Insert Buffer本身就是为了减少离散读提供性能而存在的。

 

![image-20210206120405905](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210206120405905.png)seg size代表了当前Insert Buffer的大小，free list代表了空闲列表的长度，size代表了已经合并的次数

Inserts代表了记录的插入树(对外来看的)，merged recs是合并的记录数，实际的插入数为2246304，Insert Buffer存在的问题是：在写密集的情况下，插入缓冲会占用过多的缓冲池，默认最大可以达到1/2，可以通过修改参数来对插入缓冲的大小进行控制(IBUF_POOL_SIZE_PER_MAX_SIZE)

**Channel Buffer**

可以视为Insert Buffer的升级，从1.0x引入的，可以对DML操作-INSERT, DELETE,UPDATE都进行缓冲

使用的仍然是非唯一的辅助索引，分别是Insert Buffer,Delete Buffer,Purge Buffer

对一条记录就行UPDATE操作可能分为两个过程，本质上就是比插入缓冲多了一个删除缓冲和更新缓冲。所以整体被称为Channel Buffer

- 将记录标记为已删除
- 真正将记录删除

因此Delete Buffer对应的是第一个过程，即将记录标记为已删除。

Purge Buffer对应的是第二过程，真正将记录删除

在不影响数据到前提下，InnoDB会将这些更新操作插入操作缓存到Channel Buffer之中

删除的时候就是将记录标记为已删除，不真正删除

而更新的话就会删除之前的数据

![image-20210206141634532](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210206141634532.png)

![image-20210206141647217](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210206141647217.png)

##### Insert Buffer的内部实现

Insert Buffer是一个B+树，Mysql4.1版本之前每个表都对应一个Insert Buffer B+树，4.1之后成为的全局的，负责对所有表的辅助索引进行Insert Buffer。这这课树被放置在共享表空间，默认也就是ibdata1,因此试图通过独立表空间ibd文件恢复表中数据时，往往会失败。这是因为独立表的辅助索引中的数据可能还在Insert Buffer中还未合并，所有需要先合并再恢复数据

该树是一颗B+树，因此由其叶节点和非叶子节点组成，非叶子节点存放的是查询的键值(search key)

![image-20210206143511430](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210206143511430.png)

space表示待插入记录所在表的表空间id，每个表都有一个唯一的id，可以通过space id查询到得知是哪张表

marker1字节用来兼容旧版本的Insert bUFFER，offset表示页所在的偏移量。总共9个字节。当一个辅助索引要插入到页中，如果这页不在缓冲池中，则先构造一个serach key 

![image-20210206143742690](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210206143742690.png)

![image-20210206144721221](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210206144721221.png)

在之后，第五列就是实际插入记录的各个字段了，因此教之前的原插入记录，会额外多13个字节记录一些东西

因为Insert Buffer索引后，辅助索引页中的记录可能被插入到InsertBuffer B+树中，为了保证每次合并InsertBuffer成功，还需要用一个特殊的页来标记每个辅助索引页的可用空间，这个页的类型为Insert Buffer BitMap。

每个map可以跟踪16384个辅助索引页，也就是256个区，每个InsertBufferBitMap都在16384第二个页中。

每个辅助索引页占4位

![image-20210206145205515](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210206145205515.png)

每个索引页在该bItMap页中占四位，并且存了许多信息比如索引页的剩余空间等等

##### Merge Insert Buffer

什么时候将InsertBuffer中的记录合并到真正的辅助索引页中呢？

- **辅助索引页被读取到缓冲池中，会将对应的索引页上面的插入缓冲合并到该辅助索引页**
- **Insert Buffer BitMap页追踪到该辅助索引页已无可用空间时**
- **Master Thread**

第一种情况为当辅助索引被读入缓冲池时，需要检查Insert Buffer Bitmap页，确认该辅助索引页是否有记录存放在Insert Buffer B+树中，如果有，则将Insert Buffer B+树中该页的所有记录一次性合并到该辅助索引页中，因此性能会有大幅提高。索引页被读入缓冲池中，会判断当前页是否有记录存放在Insert Buffer B+树中还未插入，如果有，则将该页的所有记录插入到辅助索引页

第二种情况，当插入辅助索引记录时检测到插入记录后可用空间会小于该辅助索引页的1/32时，会强制进行一次合并操作，将Insert Buffer B+树中该页的记录及待插入记录都插入到该辅助索引页中。可能用于页的重用？（猜测,书上未说明原因），为什么即将满的页要让其更快速的满？

第三种情况，Master Thread线程中每秒或每10秒会进行一次Merge Insert Buffer操作，不同的是，在Master Thread中进行的merge操作是根据srv_innodb_io_capacity的百分比决定要合并的辅助索引页数量。在进行merge时，如果进行merge的表已经被删除，此时可以直接丢弃已经被Insert/Change Buffer的数据记录。

#### 两次写

InsertBuffer带来的是存储引擎性能上的提升，那么doublewrite带来的是数据的可靠性

当发生数据库宕机的时候，可能正在写入某个页到某个表，而这个页只写了一部分，比如16kb的页只写了前4KB，之后就发生了宕机。

虽然可以通过重做日志来进行恢复，但是重做日志恢复的是对该页的物理操作，比如偏移量800写入aaaaa，但是如果该页本身已经发生了损坏呢？所以说，使用重做日志恢复前如果页损坏，需要一个对于该页的副本，先还原页然后再还原数据。这就是两次写。

**doublewrite由两部分组成，一部分为内存中的doublewrite，一部分为磁盘中的共享表空间连续128个页，都是2MB大小。在对缓冲池的脏页进行刷新的时候，首先写入内存中的doublewrite，然后再将数据写入的共享表，然后同时同步数据到磁盘**

写入->然后缓存到缓冲池，脏页刷新->doublewrite->再同步到磁盘

![image-20210206152354906](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210206152354906.png)

**由于innodb page是16K，一般系统page是4k,当有个update语句需要对业内记录加1，当第一个4k中记录加1后，系统宕机，重启恢复时候，innodb 不知道从哪里给记录加1，如果给16k里所有记录都加1，就会导致第一个4k里面记录加2，必然导致数据不一致，这个时候double write buffer就解决了这个问题。**	

 **Double Write的思路很简单:**

 **A. 在覆盖磁盘上的数据前，先将Page的内容写入到磁盘上的其他地方(InnoDB存储引擎中的doublewrite buffer，这里的buffer不是内存空间，是持久存储上的空间).**

 **B. 然后再将Page的内容覆盖到磁盘上原来的数据。**

 **如果在A步骤时系统故障，原来的数据没有被覆盖，还是完整的。
 如果在B步骤时系统故障，原来的数据不完整了，但是新数据已经被完整的写入了doublewrite buffer. 因此系统恢复时就可以用doublewrite buffer中的新Page来覆盖这个不完整的page。**

#### 自适应哈希索引

InnoDB存储引擎会监控表的各索引页的查询，如果观察到可以建立哈希索引来提升速度，就建立。要求就是访问模式一样

![image-20210206153821960](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210206153821960.png)

比如说频繁根据Where a= xxx查询100次 页通过该模式被访问了N次

#### 异步IO

发送完请求，不用等待立刻发送其他请求AIO，异步非阻塞IO，等待结果的到来就行

还有一个优势，就是IO Merge操作，例如用户需要访问的页是连续的，则可以直接取出多个页

![image-20210206154503259](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210206154503259.png)

提供了内核级别AIO的支持，脏页刷新等等都用到了AIO

#### 刷新领接页

当刷新一个脏页的时候，会检查该页所在区的所有页，若也有脏页，那么一起进行刷新操作

![image-20210206154656568](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210206154656568.png)

但是如果重复变脏页呢？ 并且固态硬盘的磁盘读写快，建议关闭

#### 启动丶关闭和恢复

![image-20210206154827629](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210206154827629.png)

默认就是不会将所有的脏页刷新回磁盘，和合并插入缓冲，而是选择一部分脏页刷新回磁盘#

#### undo

**undo保证了Mysql事务中的原子性**

相当于操作任何数据时都先备份数据到undo log并且记录当前数据的版本号，如果用户执行了回滚或者出错，则可以通过该数据来还原，如果想使用MVCC，去读版本号判断

undo并且可以实现MVCC 实现多版本并发控制，去判断当前数据的版本号是否大于当前事务的版本号，如果是则取读取该数据的快照，因为可重复读隔离级别下就相当于快照读，所以去undo页读取数据来保证数据的一致性，从而达到非一致性锁定，提高并发性能，为什么该事务隔离级别插入数据后可以查看到插入的数据呢？ 这是因为新增数据不存在版本号，所以也无法避免

```java
 假设有A、B两个数据，值分别为1,2。 进行+2的事务操作。

 A.事务开始.  过程中都有对应的版本号用于判断MVCC 实现乐观锁

 B.记录A=1到undo log.

 C.修改A=3.

 D.记录B=2到undo log.

 E.修改B=4.

 F.将undo log写到磁盘。

 G.将数据写到磁盘。

 H.事务提交
```

![img](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/@2O58R_1ZKJE70NW}U4%@_P.png)

行锁避免其他连接修改数据

间隙锁避免插入数据

MVCC实现可重复读通过多版本并发控制