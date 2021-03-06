---
title: mysql基础
date: 2020-05-02 09:06:48
tags: mysql
categories: Mysql
---
### mysql逻辑架构

![images](/images/mysql-struct.png)
<!--more-->

#### 连接器
连接器负责和客户端建立连接，获取权限，维持和管理连接。连接命令：mysql -h$ip -P$port -u$user -p

#### 查询缓存
mysql会缓存一些查询，下次查询如果命中就可以直接返回。但是我们不建议使用查询缓存，因为对一个表的任何更新操作都会使该表的缓存失效。mysql8.0已经删掉这个功能了。

#### 分析器
分析器对sql语句进行解析。首先是词法分析：将关键字以及表名列名等识别出来；之后是语法分析：判断这个sql语句是否满足mysql语法。

#### 优化器
优化器一般是在有多个索引时，决定使用哪个索引，或者关键查询时，决定连接顺序；

#### 执行器
执行器是调用存储引擎接口来执行语句获取结果。执行之前会判断对该表是否有权限，如果有权限，就打开表，根据存储引擎提供的接口，扫描数据行获取结果集返回。

-----------------

### redo log和binlog


**WAL（Write-Ahead Logging）：**先写日志，再写磁盘。mysql更新数据时，会写入redo log和binlog，并更新内存，这样一个更新就算完成了，等系统空闲或者需要的时候再将内存中数据刷回磁盘。虽然写redo log和binlog也会刷盘，但是是顺序写，效率很高。

**redo log：**innodb特有的，物理日志，记录的是某个页上修改了什么内容。redolog是固定大小的（这个大小可以配置），所以是循环写入的。write pos记录的是redolog中当前写入的位置，check point记录的是要擦除的位置。每次擦除时需要将数据刷到磁盘中。redolog保证了mysql异常重启，之前提交的记录也不会丢失，这个能力成为crash-safe。

**binlog：**mysql server层提供的。主要作用是主从复制以及恢复数据。binlog有三种形式：statement，row，mixed。statement格式记录的就是sql语句，row格式会记录行的内容，记两条，更新前后都有。mixed其实就是前面两种的结合，mysql会根据不同场景选用前面的两种格式。

**redolog和binlog的不同点：**

1. redolog是innodb引擎特有的，binlog是server层提供的，所有引擎都可以使用。
2.  redolog是物理日志，记录的是某个数据页做了什么修改。binlog是逻辑日志，记录的是这个语句的原始逻辑。
3. redolog是循环写的，写完了又会从头开始写。binlog是追加写，写完一个文件之后会重新开一个文件写入。


![images](/images/mysql-update.png)

根据上图我们介绍下mysql innodb引擎更新过程：

1. 首先根据条件取出要更新的行，如果在内存中，则直接返回，不再内存中则从磁盘中读取再返回。
2. 对数据进行更新，然后更新到内存。
3. redo log和binlog的写入采用两阶段提交。首先innodb引擎写入redolog，并且处于prepare阶段。
4. 然后server层执行器写binlog，并把binlog写入磁盘。
5. innodb将之前的prepare状态的redolog提交，更新完成。


这里redolog和binlog采用两阶段提交，可以保证一致性。假设数据库宕机，当数据库恢复后。遍历prepare状态的redolog，根据事务id找binlog，如果binlog没有写入，则回滚这条redolog，如果binlog已经写入，则提交这条redolog。

--------------

### 事务与事务隔离级别
**事务四个特性：**ACID（原子性，一致性，隔离性，持久性）

**隔离级别：**当数据库有多个事务同时执行时，就会有可能出现脏读，不可重复读，幻读等问题。为了解决这些问题，就需要引入隔离级别的概念。

1. 读未提交（read-uncommitted）：一个事务可以读到其他事务未提交的数据。
2. 读提交（read-committed）：一个事务只能读取别的事务已经提交的数据。
3. 可重复读（repeatable-read）：一个事务执行过程中看到的数据，总和事务启动时的一致。
4. 串行化（serializable）：事务都是串行执行，事务在读写一行数据时会加锁，其他事务只能等待获取锁。

上面四个重点理解中间两个隔离级别就行。读提交级别会读取到别的事务已经提交的数据，这样就有可能造成前后两次读取的数据不一致，造成不可重复读问题。可重复读保证了事务中读取到的数据一致，

#### 事务隔离实现
在实现上，数据库会创建一个视图，在访问数据时以视图为准。在可重复读级别下，这个视图是在事务启动时创建的，整个事务过程中都会使用这个视图。在读提交级别下，这个视图是在每个sql语句执行时创建的。

在mysql中，数据库会为每条sql更新语句记录一条对应的undo log。通过undo log可以找到前面版本的数据，这样同一条记录在数据库中就存在多个版本，这就是数据库的多版本并发控制（MVCC）。可重复读会使用事务开启时数据的read-view版本，读提交使用sql执行时的数据read-view版本。

当系统中没有比undo log更早的read-view时，这时候undo log会删除。所以我们程序中尽量不要使用长事务，否则undo log有可能保存很久。

-----


### 索引
#### 索引常见模型：

**1.哈希表：**适合只有等值查询的场景

**2.有序数组：**在等值查询查询和范围查询场景中性能都不错，但是更新成本太高，只适合静态存储引擎。

**3.搜索树：**二叉搜索树在查询和更新都是O(logN)，但是二叉搜索树在存储大量数据时，树的高度会很高，而且索引数据不可能全部存在内存中，需要存到磁盘中，这个时候就会导致多次磁盘IO。所以数据库索引一般使用N叉搜索树，也就是B+树。


#### Innodb索引模型

Innodb使用B+树存储索引，B+树中非叶子节点存储的是索引，所有数据都是存储在最下面一层的叶子节点。

Innodb数据是以页为单位存储的，一个页大小默认是16K。按照bigint的8字节，加上指针6字节，一个索引页存储的索引树大概是16*1024/（8+6）=1170。按照一行数据1k，一个数据页大概可以存储16行数据。这样一颗高度为3的B+树就可以存储1170*1170*16（2千多万）的数据，这样访问一条数据最多也就需要3次磁盘IO，所以B+ 树能够很好地配合磁盘的读写特性，减少单次查询的磁盘访问次数。

在插入和删除时，为了维护索引的有序，需要对B+树进行维护。具体插入删除操作就不介绍了。这个操作具体到页上就是当一个页满了之后，需要申请新的页，将数据移动到新的页，这个叫做页分裂，会影响到性能，同时也会影响页的利用率。当然有分裂就有合并，删除数据时，相邻两个页如果页利用率很低，就会进行合并操作。基于上面的维护过程，我们一般建议数据库表主键定义为自增主键，这样每次新增一条记录时，都是追加操作，不会涉及挪动其他数据。

#### 索引分类

  基于叶子存储的数据可以分成主键索引和非主键索引。主键索引叶子节点存储的是整行数据，非主键索引存储的是索引和主键的对应关系。通过主键索引查询数据时直接在主键索  引树上搜索就可以了。通过非主键索引查询数据时，由于非主键索引树存储的只是关键字和主键的关系，所以需要再到主键索引树上搜索数据，这个过程就称为回表。

上面我们知道通过非主键索引查询时一般需要回表，那有没有可能不需要回表。答案就是覆盖索引，如果我们查询的字段是非主键索引包含的字段或者主键时，这个时候就不需要回表，直接返回。由于覆盖索引可以减少树的搜索次数，显著提升查询性能，所以使用覆盖索引是一个常用的性能优化手段。

#### 最左前缀原则

如果查询是根据联合索引的最左N个字段或者是根据字符串索引的最左M个字符来查询，这个时候查询就能根据最左前缀原则使用索引来查询。
          
联合索引中，如果中间某个字段是通过in或者>、<等来查询，则后面的字段使用不上索引了。如下：有一个联合索引index(a,b,c)，只能使用索引上的ab字段。select * from t where a=1 and b>2 and c=3;



#### 索引下推

就拿上面的例子来说，由于只能使用索引的ab字段，假如根a=1 and b>2的条件查询到100条数据，其中c=3的有20条，c!=3的有80条。在mysql5.6之前，只能将这100条数据回表，在主键索引上一条条查询，判断c是否等于3。而在mysql5.6之后，引入了索引下推的优化，可以在遍历联合索引树中就会对c!=3的数据进行过滤，过滤之后剩20条，再对这20条进行回表查询。

-----
### 锁

根据加锁范围，mysql的锁可以分成三类：全局锁，表级锁，行级锁。



#### 全局锁

全局锁就是对整个数据库实例加锁，mysql提供一个加全局读锁的方法，命令是Flush table with read lock（FTWRL）。使用这个命令之后，数据库更新语句（增删改），数据库定义语句（建表修改表等）和更新类事务提交语句都会被阻塞。

全局锁使用的典型场景就是做全库数据备份。使用FTWRL是防止备份过程中数据被修改了，但是我们使用innodb时，由于事务隔离级别时可重复读，我们可以在备份时开启一个事务，这样就有一个全局一致性的视图，后面的更新对备份操作没有影响。具体命令就是使用mysqldump ，带上参数–single-transaction。


#### 表级锁

表级锁分为两种，一种是表锁，一种是元数据锁（meta data lock，MDL）。表锁一般就是lock tables ... read/rewrite 。不过innodb支持行级锁，我们很少在innodb中会用到表锁。

另一种表级锁是MDL。MDL不需要显式使用，在对一个表进行增删改查时，会默认加上MDL读锁，对表结构进行修改时，会默认加上MDL写锁。读锁之间不互斥，读写锁或者写锁之间会互斥。

#### 行级锁

在innodb事务中，行锁是在需要的时候加上的，但是并不是使用完就立马释放，而是在事务结束之后才会释放，这就是两阶段锁协议。鉴于此，我们在事务中需要锁住多个行时，要把最有可能造成锁冲突，最有可能影响并发度的加锁操作放在最后。比如下单操作：1、生成订单；2、扣减账户金额；3、扣减商品库存。由于第三步最有可能造成锁冲突，我们需要把这个操作放在最后，减少加锁时间。


#### 死锁和死锁检测

当并发系统中不同线程出现循环资源依赖，涉及的线程都在等待别的线程释放资源时，就会导致这几个线程都进入无限等待的状态，称为死锁。

数据库出现死锁后，一般有两种策略：1、一个是直接等待，直到超时，这个超时时间是通过innodb_lock_wait_timeout参数设置，默认是50s；2、发起死锁检测，发现死锁后，主动回滚链条中的某个事务，让其他事务继续进行。将参数 innodb_deadlock_detect 设置为 on，表示开启这个逻辑。

主动死锁检测在发生死锁的时候能快速检测到死锁并处理，但是它也是有负担的。每个新来的被堵住的线程，都会判断会不会由于自己的加入导致死锁，这是一个时间复杂度O(n)的操作，假设有1000个并发同时来更新一行数据，那么死锁检测操作的量级就是100万，虽然最后检测是没有死锁，这期间也耗费了大量CPU资源。