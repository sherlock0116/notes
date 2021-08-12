# 【HiveOperation】MSCK



最近在 HiveServer2 的日志中使用 HiveHook 抓取到很多: MSCK 的 HiveOperation.

一直不知道 MSCK 是什么操作, 于是 Google 查询:

> 基础的hive命令: `MSCK REPAIR TABLE`。

###### `MSCK REPAIR TABLE ` 命令主要是用来解决通过 hdfs dfs -put 或者 hdfs api 写入 hive 分区表的数据在hive 中无法被查询到的问题。

我们知道 hive 有个服务叫 metastore，这个服务主要是存储一些元数据信息，比如数据库名，表名或者表的分区等等信息。如果不是通过 hive 的 insert 等插入语句，很多分区信息在 metastore 中是没有的，如果插入分区数据量很多的话，你用 `ALTER TABLE table_name ADD PARTITION` 一个个分区添加十分麻烦。这时候 `MSCK REPAIR TABLE` 就派上用场了。只需要运行 `MSCK REPAIR TABLE` 命令，hive 就会去检测这个表在 hdfs 上的文件，把没有写入 metastore 的分区信息写入 metastore。

## 例子

我们先创建一个分区表，然后往其中的一个分区插入一条数据，在查看分区信息

```sql
CREATE TABLE repair_test (col_a STRING) PARTITIONED BY (par STRING);
INSERT INTO TABLE repair_test PARTITION(par="partition_1") VALUES ("test");
SHOW PARTITIONS repair_test;
```

