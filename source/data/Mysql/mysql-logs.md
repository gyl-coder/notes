[TOC]

MySQL Innodb中跟数据持久性、一致性有关的日志，有以下几种：

- Bin Log:是mysql服务层产生的日志，常用来进行数据恢复、数据库复制，常见的mysql主从架构，就是采用slave同步master的binlog实现的
- Redo Log:记录了数据操作在物理层面的修改，mysql中使用了大量缓存，修改操作时会直接修改内存，而不是立刻修改磁盘，事务进行中时会不断的产生redo log，在事务提交时进行一次flush操作，保存到磁盘中。当数据库或主机失效重启时，会根据redo log进行数据的恢复，如果redo log中有事务提交，则进行事务提交修改数据。
- Undo Log: 除了记录redo log外，当进行数据修改时还会记录undo log，undo log用于数据的撤回操作，它记录了修改的反向操作，比如，插入对应删除，修改对应修改为原来的数据，通过undo log可以实现事务回滚，并且可以根据undo log回溯到某个特定的版本的数据，实现MVCC

## Undo log

重做日志记录了事务的行为，通过它可以对数据页进行重做操作。但是事务有时还需要进行回滚，这时就需要记录回滚前的状态，比如插入一条记录，那么至少要把这条记录的主键值记录下来；修改一条记录，至少要把修改前的旧值记录下来。因此在对数据库进行修改时，InnoDB 存储引擎不但会产生 redo，还会产生一定量的 undo。这样如果用户执行的事务或者语句由于某种原因失败了，又或者用户使用 ROLLBACK 语句请求回滚，就可以利用这些 undo 信息将数据回滚到修改之前的状态。

**Undo Log：**数据库事务开始之前，会将要修改的记录存放到 Undo 日志里，当事务回滚时或者数 据库崩溃时，可以利用 Undo 日志，撤销未提交事务对数据库产生的影响。（每条数据变更(insert/update/delete)操作都伴随一条undo log的生成,并且回滚日志必须先于数据持久化到磁盘上）

**Undo Log产生和销毁：**Undo Log在事务开始前产生；事务在提交时，并不会立刻删除undo log，innodb会将该事务对应的undo log放入到删除列表中，后面会通过后台线程purge thread进 行回收处理。Undo Log属于逻辑日志，记录一个变化过程。

Undo Log存储：undo log采用段的方式管理和记录。在innodb数据文件中包含一种rollback segment回滚段，内部包含1024个undo log segment。可以通过下面一组参数来控制Undo log存 储。

```
show variables like '%innodb_undo%';
```

**Undo Log作用** 

- **事务回滚：**实现事务的原子性 Undo Log 是为了实现事务的原子性而出现的产物。事务处理过程中，如果出现了错误或者用户执 行了 ROLLBACK 语句，MySQL 可以利用 Undo Log 中的备份将数据恢复到事务开始之前的状态。 
- **实现多版本并发控制（MVCC）** Undo Log 在 MySQL InnoDB 存储引擎中用来实现多版本并发控制。事务未提交之前，Undo Log 保存了未提交之前的版本数据，Undo Log 中的数据可作为数据旧版本快照供其他并发事务进行快 照读。

**Purge清理操作**

http://mysql.taobao.org/monthly/2015/04/01/

## Redo Log

Redo：顾名思义就是重做。以恢复操作为目的，在数据库发生意外时重现操作。 

Redo Log：指事务中修改的任何数据，将最新的数据备份存储的位置（Redo Log），被称为重做日志。 

Redo Log 的生成和释放：随着事务操作的执行，就会生成Redo Log，在事务提交时会将产生 Redo Log写入Log Buffer，并不是随着事务的提交就立刻写入磁盘文件（由参数控制）。等事务操作的脏页写入 到磁盘之后，Redo Log 的使命也就完成了，Redo Log占用的空间就可以重用（被覆盖写入）。

 Redo Log工作原理：Redo Log 是为了实现事务的持久性而出现的产物。防止在发生故障的时间点，尚有脏页未写入表 的 IBD 文件中，在重启 MySQL 服务的时候，根据 Redo Log 进行重做，从而达到事务的未入磁盘 数据进行持久化这一特性。





### 既然redo log也需要存储，也涉及磁盘IO为啥还用它？

- redo log 的存储是**追加操作，属于顺序IO。**而缓存同步（刷磁盘，赃刷）是随机操作。

- 刷脏是以数据页（Page）为单位的，MySQL默认页大小是16KB，一个Page上一个小修改都要整页写入；而redo log中只包含真正需要写入的部分，无效IO大大减少。





## Checkpoint 机制

### 出现的原因

Mysql中，用户的修改操作会导致innoDB缓冲池出现脏页，如果说每次修改都需要将脏页刷新到磁盘的话，那么这个开销是非常大的。

还有一点，如果将脏页刷新到磁盘时发生了宕机，那么这个数据就无法恢复了，因此innoDB采用了Write Ahead Log策略，事务提交时，先记录到redo log，再修改页，修改完会先缓存到innoDB缓冲池中，通过后台线程去统一刷新。这样既能保证数据安全，又能很大提升性能。

但是，如果redo log无限增大（假设数据库已经运行了好几年的时间），缓冲池页无限大，那么宕机的时候通过整个redo log来恢复数据，这时整个redo log重放恢复的时间将会非常久。

Checkpoint（检查点）技术就来解决以上问题的。

### 解决的问题

- **缩短数据库恢复的时间**

当数据库发生宕机时，不需要重放整个redo log，只需要重放Checkpoint之后的redo log，这样就可以大大缩短恢复时间；

- **缓冲池不够用，将脏页刷新到磁盘**

缓冲池不够用时，根据LRU算法会淘汰掉最近最少使用的页，如果该页是脏页的话，会强制执行CheckPoint，将该脏页刷回磁盘；

- **redo log不可用时，将脏页刷新到磁盘**

由于重做日志不可能无限大，innoDB重做日志的设计按照循环使用规则，已经不再需要的日志可以被覆盖。因此如果重做日志经过一圈写到当前Checkpoint的位置，导致redo log不可用，将强制执行CheckPoint，将缓冲池的页至少刷新到当前重做日志的位置。

### 原理

CheckPoint所做的事情无非就是将缓冲池的脏页刷新回磁盘，并且每次会记录版本。这样当数据库宕机时，就知道上次CheckPoint的地方在哪，只要恢复到那个状态就行

InnoDB存储引擎是通过`LSN`（Log Sequnce Number）来标记版本的。LSN是8个字节的整形，每个页有LSN（重做日志有LSN，Checkpoint也有LSN）。存放位置：

- redo log：每次事务提交，先在log buffer生成LSN，然后刷新到磁盘的redo log，递增LSN；
- 缓冲池中的页：保存两个LSN，`第一次修改时LSN`，`最后一次修改时LSN`；每次修改页，递增`最后一次修改时LSN`；
- Checkpoint：每次将页刷新到磁盘，保存该页的`第一次修改时LSN`；

因此，数据库恢复时，只需要重放redo log的LSN - Checkpoint的LSN之间的操作。

**Checkpoint类型分为：**

- Sharp Checkpoint：发生在数据库关闭时将所有的脏页都刷新回磁盘，如果此时数据库脏页非常多，则会很大影响性能；
- Fuzzy Checkpoint：默认方式，只刷新一部分脏页，不是刷新所有脏页；主要有以下几种情况：
  - Master Thread Checkpoint：每秒或者10秒的频率异步刷新缓冲池的脏页到磁盘。（由Page Cleaner Thread完成）
  - FLUSH_LRU_LIST Checkpoint：缓冲池不够用时，根据LRU算法会淘汰掉最近最少使用的页，如果该页是脏页的话，会强制执行CheckPoint，将该脏页刷回磁盘（由Page Cleaner Thread完成）；
  - Async/Sync Flush Checkpoint：重做日志不可用的情况，需要强制从脏页列表中选取一些脏页刷新磁盘到缓存（由Page Cleaner Thread完成）。（Dirty Page too much Checkpoint：即脏页数量太多，导致强制进行Checkpoint。由参数`innodb_max_dirty_pages_pt`来控制，默认75（即75%）。当脏页数量占据75%缓冲池时，刷新一部分脏页到磁盘。）

## 慢查询日志

https://zhuanlan.zhihu.com/p/58011817

 MySQL的慢查询日志是MySQL提供的一种日志记录，它用来记录在MySQL中响应时间超过阀值的语句，具体指运行时间超过long_query_time值的SQL，则会被记录到慢查询日志中。long_query_time的默认值为10，意思是运行10S以上的语句。默认情况下，Mysql数据库并不启动慢查询日志，需要我们手动来设置这个参数，当然，如果不是调优需要的话，一般不建议启动该参数，因为开启慢查询日志会或多或少带来一定的性能影响。慢查询日志支持将日志记录写入文件，也支持将日志记录写入数据库表。

### Mysql 什么情况会造成慢查

- 网络，各个节点之间的网络
- OS系统 ，即数据库服务器
- MySQL数据库本身

1. 网络层面
   1. 网络丢包，重传
   2. 网卡满，比如大字段
   3. 网络链路长
2. 受IO影响
   1. 磁盘io被其他任务占用
   2. RAID 卡 充放电，重置
   3. io调度算法
3. cpu 类型
4. 数据库层面
   1. **没有索引，或者索引使用不正确**
   2. 隐式转换
   3. 执行计划错误
   4. **数据量大**
   5. MetaData Lock 锁等待
   6. 并发更新同一行记录
   7. **数据分布不均匀**
   8. sql姿势不合理
   9. 表结构设计问题
   10. **innodb 刷脏页**
   11. undo 没有被purge回收

https://www.infoq.cn/article/xw54gb1oe6fpxiggch19



### 如何查看慢查询

开启慢查询日志
mysql> set global slow_query_log=1;
定义时间SQL查询的超时时间
mysql> set global long_query_time = 0.005;
查看慢查询日志的保存路径
mysql> show global variables like ‘slow_query_log_file’;
查看慢查询
cat /var/log/mysql/slow.log

### 如何处理慢查询

​       sql及索引优化。 

​       数据库表结构，根据数据设计查询最优的表结构。

​      系统配置优化，对打开文件数和安全的限制。 

​      硬件，选择最适合数据库的cpu，更快的IO,更大的内存，cpu不是越多越好， IO并不能减少锁的机制，也就是不能减少阻塞，所以说硬件的优化成本越高，效果最差。