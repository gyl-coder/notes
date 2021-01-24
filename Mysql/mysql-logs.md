https://zhuanlan.zhihu.com/p/58011817



## 慢查询日志

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