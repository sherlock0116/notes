# 【Doris】数据模型及 rollup

## 一、前言

Apache Doris 是一个现代化的 MPP（大规模并行分析）分析型数据库产品。仅需亚秒级响应时间即可获得查询结果，有效地支持实时数据分析。Apache Doris 的分布式架构非常简洁，易于运维，并且可以支持 10PB 以上的超大数据集。

在 Doris 中，数据以表（Table）的形式进行逻辑上的描述。一张表包括行（Row）和列（Column）。
Row 即用户的一行数据。Column 用于描述一行数据中不同的字段。

Doris 中将 Column 可以分为两大类：Key 和 Value。从业务角度看，Key 和 Value 可以分别对应维度列和指标列。

指标列的运算类型: MAX, MIN, SUM, REPLACE

## 二、数据模型

Doris 的数据模型主要分为3类:

- Aggregate
- Uniq
- Duplicate

### 2.1 Aggregate 模型

Aggregate 模型, 提前聚合数据, 适合报表和多维分析业务。

- SUM：求和，多行的 Value 进行累加。
- REPLACE：替代，下一批数据中的 Value 会替换之前导入过的行中的 Value。
- MAX：保留最大值。
- MIN：保留最小值。

```sql
CREATE TABLE IF NOT EXISTS example_db.expamle_tbl(
  `user_id` LARGEINT NOT NULL COMMENT "用户id",
  `date` DATE NOT NULL COMMENT "数据灌入日期时间",
  `city` VARCHAR(20) COMMENT "用户所在城市",
  `age` SMALLINT COMMENT "用户年龄",
  `sex` TINYINT COMMENT "用户性别",
  `last_visit_date` DATETIME REPLACE DEFAULT "1970-01-01 00:00:00" COMMENT "用户最后一次访问时间",
  `cost` BIGINT SUM DEFAULT "0" COMMENT "用户总消费",
  `max_dwell_time` INT MAX DEFAULT "0" COMMENT "用户最大停留时间",
  `min_dwell_time` INT MIN DEFAULT "99999" COMMENT "用户最小停留时间"
)
AGGREGATE KEY(`user_id`, `date`, `city`, `age`, `sex`)
... /* 省略 Partition 和 Distribution 信息 */
；
```

| user_id | date       | city | age  | sex  | last_visit_date     | cost | max_dwell_time | min_dwell_time |
| ------- | ---------- | ---- | ---- | ---- | ------------------- | ---- | -------------- | -------------- |
| 10000   | 2017-10-01 | 北京 | 20   | 0    | 2017-10-01 06:00:00 | 20   | 10             | 10             |
| 10000   | 2017-10-01 | 北京 | 20   | 0    | 2017-10-01 07:00:00 | 15   | 2              | 2              |

在导入上述数据时，最终表里面就只有一行数据：

| user_id | date       | city | age  | sex  | last_visit_date     | cost | max_dwell_time | min_dwell_time |
| ------- | ---------- | ---- | ---- | ---- | ------------------- | ---- | -------------- | -------------- |
| 10000   | 2017-10-01 | 北京 | 20   | 0    | 2017-10-01 07:00:00 | 35   | 10             | 2              |

其中 user_id, date, city, age, sex 因为没有聚合模型，因此只有当他们都一样时，才可以发生后面的聚合；假设 city 不一样就不能聚合；

如果业务场景就是需要存入明细，那么一般的做法是列增加时间戳，这样每一列都不一样，就可以保存数据明细；

数据在不同时间，可能聚合的程度不一致。比如一批数据刚导入时，可能还未与之前已存在的数据进行聚合。但是对于用户而言，用户只能查询到聚合后的数据。即不同的聚合程度对于用户查询而言是透明的。用户需始终认为数据以最终的完成的聚合程度存在，而不应假设某些聚合还未发生。

### 2.2 Uniq 模型

Uniq 模型, 保证 Key 的唯一性,适用于有更新需求的分析业务。

其实是 Aggregate 聚合模型的一个特例，即只有 aggregate key 是唯一的，其他列都是replace 特性。

```sql
CREATE TABLE IF NOT EXISTS example_db.expamle_tbl (
  `user_id` LARGEINT NOT NULL COMMENT "用户id",
  `username` VARCHAR(50) NOT NULL COMMENT "用户昵称",
  `city` VARCHAR(20) COMMENT "用户所在城市",
  `age` SMALLINT COMMENT "用户年龄",
  `sex` TINYINT COMMENT "用户性别",
  `phone` LARGEINT COMMENT "用户电话",
  `address` VARCHAR(500) COMMENT "用户地址",
  `register_time` DATETIME COMMENT "用户注册时间"
)
UNIQUE KEY(`user_id`, `user_name`)
... /* 省略 Partition 和 Distribution 信息 */
;
```

等同于：

```sql
CREATE TABLE IF NOT EXISTS example_db.expamle_tbl (
  `user_id` LARGEINT NOT NULL COMMENT "用户id",
  `username` VARCHAR(50) NOT NULL COMMENT "用户昵称",
  `city` VARCHAR(20) REPLACE COMMENT "用户所在城市",
  `age` SMALLINT REPLACE COMMENT "用户年龄",
  `sex` TINYINT REPLACE COMMENT "用户性别",
  `phone` LARGEINT REPLACE COMMENT "用户电话",
  `address` VARCHAR(500) REPLACE COMMENT "用户地址",
  `register_time` DATETIME REPLACE COMMENT "用户注册时间"
)
AGGREGATE KEY(`user_id`, `user_name`)
... /* 省略 Partition 和 Distribution 信息 */
;
```

### 2.3 Duplicate

Duplicate 模型, 只指定排序列，相同的行不会合并。适用于数据无需提前聚合的分析业务。

```sql
CREATE TABLE IF NOT EXISTS example_db.expamle_tbl(
  `timestamp` DATETIME NOT NULL COMMENT "日志时间",
  `type` INT NOT NULL COMMENT "日志类型",
  `error_code` INT COMMENT "错误码",
  `error_msg` VARCHAR(1024) COMMENT "错误详细信息",
  `op_id` BIGINT COMMENT "负责人id",
  `op_time` DATETIME COMMENT "处理时间"
)
DUPLICATE KEY(`timestamp`, `type`)
... /* 省略 Partition 和 Distribution 信息 */
;
```

这种数据模型区别于 Aggregate 和 Uniq 模型。数据完全按照导入文件中的数据进行存储，不会有任何聚合。即使两行数据完全相同，也都会保留。 而在建表语句中指定的 DUPLICATE KEY，只是用来指明底层数据按照那些列进行排序。

### 2.4 数据模型的选择建议

因为数据模型在建表时就已经确定，且**无法修改**。所以，选择一个合适的数据模型**非常重要**。

1. Aggregate 模型可以通过预聚合，极大地降低聚合查询时所需扫描的数据量和查询的计算量，非常适合有固定模式的报表类查询场景。但是该模型对 count(*) 查询很不友好。同时因为固定了 Value 列上的聚合方式，在进行其他类型的聚合查询时，需要考虑语意正确性。
2. Uniq 模型针对需要唯一主键约束的场景，可以保证主键唯一性约束。但是无法利用 ROLLUP 等预聚合带来的查询优势（因为本质是 REPLACE，没有 SUM 这种聚合方式）。
3. Duplicate 适合任意维度的 Ad-hoc 查询。虽然同样无法利用预聚合的特性，但是不受聚合模型的约束，可以发挥列存模型的优势（只读取相关列，而不需要读取所有 Key 列）。

## 三、ROLLUP

```sql
rollup 支持如下几种创建方式：
-- rollup 基础语法
-- 1. 在表上创建 rollup
ALTER TABLE [table_name] ADD ROLLUP [rollup_name](col1, col2, col3, ..., );
-- 2. 查看 rollup 进度
SHOW ALTER TABLE ROLLUP;
-- 3. 查看表 rollup 信息
DESC [table_name] ALL;
-- 4. 查看执行计划
EXPLAIN [sql]
-- 5. 删除 rollup
DROP ROLLUP [rollup_name]
-- 6. 修改 rollup name
RENAME ROLLUP [old_rollup_name] [new_rollup_name]
-- 注意：
不能删除 base index
```

上面的每个模型产生的其实是一个 base index 表，很多场景下我们需要在 base 表的基础上粗化维度列来建立其他表，以便于提高查询效率。

Rollup 本质上可以理解为原始表(Base Table)的一个物化索引。建立 Rollup 时可只选取 Base Table 中的部分列作为 Schema。Schema 中的字段顺序也可与 Base Table 不同

表 tmp_order_item_detail 

```sql
CREATE TABLE `tmp_order_item_detail` (
  amount_date varchar(100) NULL COMMENT "销售日期",
  amount_date_year varchar(100) NULL COMMENT "销售年份",
  amount_date_month varchar(100) NULL COMMENT "销售月份",
  amount_date_week varchar(100) NULL COMMENT "销售周",
  order_id varchar(100) NULL COMMENT "订单号",
  order_item_id varchar(100) NULL COMMENT "订单明细号",
  account_id varchar(100) NULL COMMENT "用户id",
  account_name varchar(100) NULL COMMENT "用户姓名",
  amount decimal(20, 2) SUM COMMENT "销售额"
)
ENGINE=OLAP
AGGREGATE KEY(`amount_date`, `amount_date_year`, `amount_date_month`, `amount_date_week`, `order_id`, `order_item_id`, `account_id`, `account_name`)
COMMENT "OLAP"
DISTRIBUTED BY HASH(`amount_date`) BUCKETS 4
PROPERTIES (
	"replication_num" = "3",
	"in_memory" = "false",
	"storage_format" = "DEFAULT"
);
```

因为该表创建时的聚合维度--`amount_date`, `amount_date_year`, `amount_date_month`, `amount_date_week`, `order_id`, `order_item_id`, `account_id`, `account_name`

如果当前需要统计: 年交易额, 月交易额, 周交易额, 日交易额, 这四种场景的聚合指标分别是amount_date_year, amount_date_month, amount_date_week, amount_date, 它们的维度指标都被放粗了(原表(base index) 的指标更细)

此时我们就可以借助 rollup 物化视图来进行聚合查询

### 3.1 测试场景(年交易额)

测试表: test.tmp_order_item_detail

表数据量: 11879603

```sql
mysql> select count(*) from `tmp_order_item_detail`;

+----------+
| count(*) |
+----------+
| 11879603 |
+----------+
1 row in set (14.33 sec)
```

测试表结构: 

```sql
desc `tmp_order_item_detail`;
+-------------------+---------------+------+-------+---------+-------+
| Field             | Type          | Null | Key   | Default | Extra |
+-------------------+---------------+------+-------+---------+-------+
| amount_date       | VARCHAR(100)  | Yes  | true  | N/A     |       |
| amount_date_year  | VARCHAR(100)  | Yes  | true  | N/A     |       |
| amount_date_month | VARCHAR(100)  | Yes  | true  | N/A     |       |
| amount_date_week  | VARCHAR(100)  | Yes  | true  | N/A     |       |
| order_id          | VARCHAR(100)  | Yes  | true  | N/A     |       |
| order_item_id     | VARCHAR(100)  | Yes  | true  | N/A     |       |
| account_id        | VARCHAR(100)  | Yes  | true  | N/A     |       |
| account_name      | VARCHAR(100)  | Yes  | true  | N/A     |       |
| amount            | DECIMAL(20,2) | Yes  | false | N/A     | SUM   |
+-------------------+---------------+------+-------+---------+-------+
```

1. 未创建 rollup 下的查询效率

```sql
mysql> select `amount_date_year`, sum(amount) from `tmp_order_item_detail` group by `amount_date_year` order by `amount_date_year`;
+------------------+-----------------+
| amount_date_year | sum(`amount`)   |
+------------------+-----------------+
| 2015             |   5898095889.92 |
| 2016             |  90431312096.32 |
| 2017             | 256333919101.52 |
| 2018             |  266077628875.5 |
| 2019             | 313305335295.16 |
| 2020             | 323026992161.12 |
| 2021             |  86126788482.15 |
+------------------+-----------------+
7 rows in set (2.48 sec)

mysql> DESC `tmp_order_item_detail` ALL;
+-----------------------+---------------+-------------------+---------------+------+-------+---------+-------+
| IndexName             | IndexKeysType | Field             | Type          | Null | Key   | Default | Extra |
+-----------------------+---------------+-------------------+---------------+------+-------+---------+-------+
| tmp_order_item_detail | AGG_KEYS      | amount_date       | VARCHAR(100)  | Yes  | true  | N/A     |       |
|                       |               | amount_date_year  | VARCHAR(100)  | Yes  | true  | N/A     |       |
|                       |               | amount_date_month | VARCHAR(100)  | Yes  | true  | N/A     |       |
|                       |               | amount_date_week  | VARCHAR(100)  | Yes  | true  | N/A     |       |
|                       |               | order_id          | VARCHAR(100)  | Yes  | true  | N/A     |       |
|                       |               | order_item_id     | VARCHAR(100)  | Yes  | true  | N/A     |       |
|                       |               | account_id        | VARCHAR(100)  | Yes  | true  | N/A     |       |
|                       |               | account_name      | VARCHAR(100)  | Yes  | true  | N/A     |       |
|                       |               | amount            | DECIMAL(20,2) | Yes  | false | N/A     | SUM   |
+-----------------------+---------------+-------------------+---------------+------+-------+---------+-------+
9 rows in set (0.01 sec)
```

2. 创建 rollup 后的查询效率

```sql
-- 创建 rollup
ALTER TABLE `tmp_order_item_detail` ADD ROLLUP rollup_yearly_amount(amount_date_year, amount);

-- 查看当前表上的所有 rollup
mysql> DESC `tmp_order_item_detail` ALL;
+-----------------------+---------------+-------------------+---------------+------+-------+---------+-------+
| IndexName             | IndexKeysType | Field             | Type          | Null | Key   | Default | Extra |
+-----------------------+---------------+-------------------+---------------+------+-------+---------+-------+
| tmp_order_item_detail | AGG_KEYS      | amount_date       | VARCHAR(100)  | Yes  | true  | N/A     |       |
|                       |               | amount_date_year  | VARCHAR(100)  | Yes  | true  | N/A     |       |
|                       |               | amount_date_month | VARCHAR(100)  | Yes  | true  | N/A     |       |
|                       |               | amount_date_week  | VARCHAR(100)  | Yes  | true  | N/A     |       |
|                       |               | order_id          | VARCHAR(100)  | Yes  | true  | N/A     |       |
|                       |               | order_item_id     | VARCHAR(100)  | Yes  | true  | N/A     |       |
|                       |               | account_id        | VARCHAR(100)  | Yes  | true  | N/A     |       |
|                       |               | account_name      | VARCHAR(100)  | Yes  | true  | N/A     |       |
|                       |               | amount            | DECIMAL(20,2) | Yes  | false | N/A     | SUM   |
|                       |               |                   |               |      |       |         |       |
| rollup_yearly_amount  | AGG_KEYS      | amount_date_year  | VARCHAR(100)  | Yes  | true  | N/A     |       |
|                       |               | amount            | DECIMAL(20,2) | Yes  | false | N/A     | SUM   |
+-----------------------+---------------+-------------------+---------------+------+-------+---------+-------+
12 rows in set (0.01 sec)

-- 再次执行查询
mysql> select `amount_date_year`, sum(amount) from `tmp_order_item_detail` group by `amount_date_year` order by `amount_date_year`;
+------------------+-----------------+
| amount_date_year | sum(`amount`)   |
+------------------+-----------------+
| 2015             |   5898095889.92 |
| 2016             |  90431312096.32 |
| 2017             | 256333919101.52 |
| 2018             |  266077628875.5 |
| 2019             | 313305335295.16 |
| 2020             | 323026992161.12 |
| 2021             |  86126788482.15 |
+------------------+-----------------+
7 rows in set (0.02 sec)

-- 查看逻辑执行计划, rollup 是否命中
mysql> EXPLAIN select `amount_date_year`, sum(amount) from `tmp_order_item_detail` group by `amount_date_year` order by `amount_date_year`;
+--------------------------------------------------------------------------------------+
| Explain String                                                                       |
+--------------------------------------------------------------------------------------+
| PLAN FRAGMENT 0                                                                      |
|  OUTPUT EXPRS:<slot 4> <slot 2> `amount_date_year` | <slot 5> <slot 3> sum(`amount`) |
|   PARTITION: UNPARTITIONED                                                           |
|                                                                                      |
|   RESULT SINK                                                                        |
|                                                                                      |
|   5:MERGING-EXCHANGE                                                                 |
|      limit: 65535                                                                    |
|      tuple ids: 2                                                                    |
|                                                                                      |
| PLAN FRAGMENT 1                                                                      |
|  OUTPUT EXPRS:                                                                       |
|   PARTITION: HASH_PARTITIONED: <slot 2> `amount_date_year`                           |
|                                                                                      |
|   STREAM DATA SINK                                                                   |
|     EXCHANGE ID: 05                                                                  |
|     UNPARTITIONED                                                                    |
|                                                                                      |
|   2:TOP-N                                                                            |
|   |  order by: <slot 4> <slot 2> `amount_date_year` ASC                              |
|   |  offset: 0                                                                       |
|   |  limit: 65535                                                                    |
|   |  tuple ids: 2                                                                    |
|   |                                                                                  |
|   4:AGGREGATE (merge finalize)                                                       |
|   |  output: sum(<slot 3> sum(`amount`))                                             |
|   |  group by: <slot 2> `amount_date_year`                                           |
|   |  tuple ids: 1                                                                    |
|   |                                                                                  |
|   3:EXCHANGE                                                                         |
|      tuple ids: 1                                                                    |
|                                                                                      |
| PLAN FRAGMENT 2                                                                      |
|  OUTPUT EXPRS:                                                                       |
|   PARTITION: RANDOM                                                                  |
|                                                                                      |
|   STREAM DATA SINK                                                                   |
|     EXCHANGE ID: 03                                                                  |
|     HASH_PARTITIONED: <slot 2> `amount_date_year`                                    |
|                                                                                      |
|   1:AGGREGATE (update serialize)                                                     |
|   |  STREAMING                                                                       |
|   |  output: sum(`amount`)                                                           |
|   |  group by: `amount_date_year`                                                    |
|   |  tuple ids: 1                                                                    |
|   |                                                                                  |
|   0:OlapScanNode                                                                     |
|      TABLE: tmp_order_item_detail                                                    |
|      PREAGGREGATION: ON                                                              |
|      partitions=1/1                                                                  |
|      rollup: rollup_yearly_amount                                                    |
|      tabletRatio=4/4                                                                 |
|      tabletList=40702,40706,40710,40714                                              |
|      cardinality=42                                                                  |
|      avgRowSize=66.0                                                                 |
|      numNodes=3                                                                      |
|      tuple ids: 0                                                                    |
+--------------------------------------------------------------------------------------+
57 rows in set (0.01 sec)
```

