## Mysql 索引操作

### 重建索引

重建索引在常规的数据库维护操作中经常使用。在数据库运行了较长时间后，索引都有损坏的可能，这时就需要重建。对数据重建索引可以起到提高检索效率。

```
主键索引的重建
# 正确做法
alter table T engine=InnoDB

# 错误做法
alter table T drop primary key;
alter table T add primary key(id);
不论是删除主键还是创建主键，都会将整个表重建。所以连着执行这两个语句的话，第一个语句就白做了。
直接删掉主键索引会使得所有的二级索引都失效，并且会用 ROWID 来作主键索引。

非主键索引的重建
alter table T drop index k;
alter table T add index(k);
```

#### 为什么需要重建索引？

索引可能因为删除，或者页分裂等原因，导致数据页有空洞，重建索引的过程会创建一个新的索引，把数据按顺序插入，这样页面的利用率高，也就是索引更紧凑、更省空间。



### 创建索引

索引的创建可以在CREATE TABLE语句中进行，也可以单独用CREATE INDEX或ALTER TABLE来给表增加索引。以下命令语句分别展示了如何创建主键索引（PRIMARY KEY），联合索引（UNIQUE）和普通索引（INDEX）的方法。

```
mysql> ALTER TABLE `table_name` ADD INDEX `index_name` (column list);
mysql> ALTER TABLE `table_name` ADD UNIQUE `index_name` (column list);
mysql> ALTER TABLE `table_name` ADD PRIMARY KEY `index_name` (column list);
mysql> CREATE INDEX `index_name` ON `table_name` (column_list);
mysql> CREATE UNIQUE INDEX `index_name` ON `table_name` (column_list);

例如：
mysql> ALTER TABLE `article` ADD INDEX `id`;//给article表增加id索引
或者：
mysql> ALTER TABLE `article` ADD INDEX (`id`,`order_id`); //给article表增加id索引，order_id索引
```



### 查询数据表索引

```
mysql> show index from `table_name`;
```




### 删除索引

删除索引可以使用ALTER TABLE或DROP INDEX语句来实现。DROP INDEX可以在ALTER TABLE内部作为一条语句处理，其格式如下：

```
mysql> DROP index `index_name` ON `table_name` (column list);
mysql> ALTER TABLE `table_name` DROP INDEX `index_name` (column list);
mysql> ALTER TABLE `table_name` DROP UNIQUE `index_name` (column list);
mysql> ALTER TABLE `table_name` DROP PRIMARY KEY `index_name` (column list);
```

在前面的三条语句中，都删除了table_name中的索引index_name。而在最后一条语句中，只在删除PRIMARY KEY索引中使用，因为一个表只可能有一个PRIMARY KEY索引，因此也可不指定索引名。如果没有创建PRIMARY KEY索引，但表具有一个或多个UNIQUE索引，则MySQL将删除第一个UNIQUE索引。如果从表中删除某列，则索引会受影响。对于多列组合的索引，如果删除其中的某列，则该列也会从索引中删除。如果删除组成索引的所有列，则整个索引将被删除。



