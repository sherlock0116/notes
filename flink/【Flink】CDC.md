# 【Flink】CDC

Flink1.11 引入了 CDC (Change Data Capture) 的 connector，使用CDC我们可以从数据库中获取已提交的更改并将这些更改发送到下游，供下游使用。这些变更可以包括INSERT,DELETE,UPDATE等。大大简化了数据处理的流程。

Flink1.11 的 CDC connector主要包括：`MySQL CDC` 和 `Postgres CDC`, 同时对 Kafka 的 **Connector** 支持 `canal-json` 和 `debezium-json` 以及`changelog-json` 的 format。

CDC 的使用场景：

- 数据库之间的全量/增量数据同步
- 审计日志
- 数据库之上的实时物化视图
- 基于CDC的维表join
- ...

## 一、Debezium 

Debezium 是一个分布式平台，它将现有的数据库转换为事件流，应用程序消费事件流，就可以知道数据库中的每一个行级更改，并立即做出响应。

Debezium 构建在 Apache Kafka 之上，并提供 Kafka 连接器来监视特定的数据库。在介绍 Debezium 之前，我们要先了解一下什么是 Kafka Connect。

### 1.1 Kafka Connect

Kafka 相信大家都很熟悉，是一款分布式，高性能的消息队列框架。

一般情况下，读写 Kafka 数据，都是用 Consumer 和 Producer Api 来完成，但是自己实现这些需要去考虑很多额外的东西，比如管理 Schema，容错，并行化，数据延迟，监控等等问题。

而在 0.9.0.0 版本之后，官方推出了 Kafka Connect ，大大减少了程序员的工作量，它有下面的特性：

- 统一而通用的框架；
- 支持分布式模式和单机模式；
- REST 接口，用来查看和管理Kafka connectors；
- 自动化的offset管理，开发人员不必担心错误处理的影响；
- 分布式、可扩展；
- 流/批处理集成。



Kafka Connect 有两个核心的概念：Source 和 Sink，Source 负责导入数据到 Kafka，Sink 负责从 Kafka 导出数据，它们都被称为是 Connector。

如下图，左边的 Source 负责从源数据（RDBMS，File等）读数据到 Kafka，右边的 Sinks 负责从 Kafka 消费到其他系统。

<img src="https://www.aboutyun.com/data/attachment/forum/202103/03/075539n3i62rc1sg1g92q1.png" style="zoom:67%;" />

### 1.2 Debezium 架构和实现原理

Debezium 有三种方式可以实现变化数据的捕获

1. 以插件的形式，部署在 Kafka Connect 上

   ![](https://www.aboutyun.com/data/attachment/forum/202103/03/075618vdziih5lthzlulg0.png)

   在上图中，中间的部分是 Kafka Broker，而 Kafka Connect 是单独的服务，需要下载 debezium-connector-mysql 连接器，解压到服务器指定的地方，然后在 connect-distribute.properties 中指定连接器的根路径，即可使用。

   

2. Debezium Server

   <img src="https://www.aboutyun.com/data/attachment/forum/202103/03/075726lf3ydywzy46mblyz.png" style="zoom:67%;" />

   这种模式中，需要配置不同的连接器，从源头处捕获数据的变化，序列化成指定的格式，发送到指定的系统中。

   

3. 内嵌在应用程序里

   内嵌模式，既不依赖 Kafka，也不依赖 Debezium Server，用户可以在自己的应用程序中，依赖 Debezium 的 api 自行处理获取到的数据，并同步到其他源上。Flink CDC 就是使用了该模式。

## 二、CDC 数据同步方案

CDC 全称是 Change Data Capture ，它是一个比较广义的概念，只要能捕获变更的数据，我们都可以称为 CDC 。业界主要有基于查询的 CDC 和基于日志的 CDC ，可以从下面表格对比他们功能和差异点。

<img src="https://ucc.alicdn.com/pic/developer-ecology/7806a56be96b4731892494775f393a39.jpg" style="zoom:50%;" />

经过以上对比，我们可以发现基于日志 CDC 有以下这几种优势：

- 能够捕获所有数据的变化，捕获完整的变更记录。在异地容灾，数据备份等场景中得到广泛应用，如果是基于查询的 CDC 有可能导致两次查询的中间一部分数据丢失
- 每次 DML 操作均有记录无需像查询 CDC 这样发起全表扫描进行过滤，拥有更高的效率和性能，具有低延迟，不增加数据库负载的优势
- 无需入侵业务，业务解耦，无需更改业务模型
- 捕获删除事件和捕获旧记录的状态，在查询 CDC 中，周期的查询无法感知中间数据是否删除

<img src="https://ucc.alicdn.com/pic/developer-ecology/1f5bd7fa81cd4a9ca6e21b13d1739a5a.png" style="zoom:67%;" />

### 2.1 基于日志的 CDC 同步方案

从 ETL 的角度进行分析，一般采集的都是业务库数据，这里使用 MySQL 作为需要采集的数据库，通过 Debezium 把 MySQL Binlog 进行采集后发送至 Kafka 消息队列，然后对接一些实时计算引擎或者 APP 进行消费后把数据传输入 OLAP 系统或者其他存储介质。

Flink 希望打通更多数据源，发挥完整的计算能力。我们生产中主要来源于业务日志和数据库日志，Flink 在业务日志的支持上已经非常完善，但是在数据库日志支持方面在 Flink 1.11 前还属于一片空白，这就是为什么要集成 CDC 的原因之一。

Flink SQL 内部支持了完整的 changelog 机制，所以 Flink 对接 CDC 数据只需要把CDC 数据转换成 Flink 认识的数据，所以在 Flink 1.11 里面重构了 TableSource 接口，以便更好支持和集成 CDC。

<img src="https://ucc.alicdn.com/pic/developer-ecology/5d5c18c7805c4a9c89796d6591171c04.png" style="zoom:80%;" />

<img src="https://ucc.alicdn.com/pic/developer-ecology/c3c6a01ee8604339907bacf133fab4e3.png" style="zoom:80%;" />

重构后的 TableSource 输出的都是 RowData 数据结构，代表了一行的数据。在RowData 上面会有一个元数据的信息，我们称为 RowKind 。RowKind 里面包括了插入、更新前、更新后、删除，这样和数据库里面的 binlog 概念十分类似。通过 Debezium 采集的 JSON 格式，包含了旧数据和新数据行以及原数据信息，op 的 u表示是 update 更新操作标识符，ts_ms 表示同步的时间戳。因此，对接 Debezium JSON 的数据，其实就是将这种原始的 JSON 数据转换成 Flink 认识的 RowData。

传统架构方案中就是采用第三方工具，比如 canal、debezium 及 Maxwell 等，实时采集数据库的变更日志，然后将数据发送到 kafka 等消息队列。然后再通过其他的组件，比如 flink、spark 等等来消费 kafka 的数据，计算之后发送到下游系统。整体的架构如下所示：

#### 2.1.1 选择 Flink 作为 ETL 工具

当选择 Flink 作为 ETL 工具时，在数据同步场景，如下图同步结构：

<img src="https://ucc.alicdn.com/pic/developer-ecology/768d5052ec0c4ced8d0e79f71843c869.png" style="zoom: 67%;" />

通过 Debezium 订阅业务库 MySQL 的 Binlog 传输至 Kafka ，Flink 通过创建 Kafka 表指定 format 格式为 debezium-json ，然后通过 Flink 进行计算后或者直接插入到其他外部数据存储系统，例如图中的 Elasticsearch 和 PostgreSQL。

<img src="https://ucc.alicdn.com/pic/developer-ecology/ea195acd250d491ca9e55e67be428006.png" style="zoom:67%;" />

但是这个架构有个缺点，我们可以看到采集端组件过多导致维护繁杂，这时候就会想是否可以用 Flink SQL 直接对接 MySQL 的 binlog 数据呢，有没可以替代的方案呢？

答案是有的！经过改进后结构如下图：

<img src="https://ucc.alicdn.com/pic/developer-ecology/6231a71c5b78493783bc0f3c024d9e65.png" style="zoom:67%;" />

社区开发了 flink-cdc-connectors 组件，这是一个可以直接从 MySQL、PostgreSQL 等数据库直接读取全量数据和增量变更数据的 source 组件。目前也已开源，开源地址：

> https://github.com/ververica/flink-cdc-connectors

flink-cdc-connectors 可以用来替换 Debezium+Kafka 的数据采集模块，从而实现 Flink SQL 采集+计算+传输（ETL）一体化，这样做的优点有以下：

· 开箱即用，简单易上手
· 减少维护的组件，简化实时链路，减轻部署成本
· 减小端到端延迟
· Flink 自身支持 Exactly Once 的读取和计算
· 数据不落地，减少存储成本
· 支持全量和增量流式读取
· binlog 采集位点可回溯*

##### 2.1.1.1 canal format

在国内，用的比较多的是阿里巴巴开源的canal，我们可以使用canal订阅mysql的binlog日志，canal会将mysql库的变更数据组织成它固定的JSON或protobuf 格式发到kafka，以供下游使用。

canal解析后的json数据格式如下：

```json
{
  "data": [
    {
      "id": "111",
      "name": "scooter",
      "description": "Big 2-wheel scooter",
      "weight": "5.18"
    }
  ],
  "database": "inventory",
  "es": 1589373560000,
  "id": 9,
  "isDdl": false,
  "mysqlType": {
    "id": "INTEGER",
    "name": "VARCHAR(255)",
    "description": "VARCHAR(512)",
    "weight": "FLOAT"
  },
  "old": [
    {
      "weight": "5.15"
    }
  ],
  "pkNames": [
    "id"
  ],
  "sql": "",
  "sqlType": {
    "id": 4,
    "name": 12,
    "description": 12,
    "weight": 7
  },
  "table": "products",
  "ts": 1589373560798,
  "type": "UPDATE"
}
```

简单讲下几个核心的字段:

- type : 描述操作的类型，包括‘UPDATE’, 'INSERT', 'DELETE'。
- data : 代表操作的数据。如果为'INSERT'，则表示行的内容；如果为'UPDATE'，则表示行的更新后的状态；如果为'DELETE'，则表示删除前的状态。
- old ：可选字段，如果存在，则表示更新之前的内容，如果不是update操作，则为 null。

完整的语义如下;

```java
private String                    destination;           // 对应canal的实例或者MQ的topic
private String                    groupId;               // 对应mq的group id
private String                    database;              // 数据库或schema
private String                    table;                 // 表名
private List<String>              pkNames;
private Boolean                   isDdl;
private String                    type;                  // 类型: INSERT UPDATE DELETE
// binlog executeTime
private Long                      es;                    // 执行耗时
// dml build timeStamp
private Long                      ts;                    // 同步时间
private String                    sql;                   // 执行的sql, dml sql为空
private List<Map<String, Object>> data;                  // 数据列表
private List<Map<String, Object>> old;                   // 旧数据列表, 用于update, size和data的size一一对应
```

在 flink sql 中，消费这个数据的 sql 如下：

```sql
CREATE TABLE topic_products (
  id BIGINT,
  name STRING,
  description STRING,
  weight DECIMAL(10, 2)
) WITH (
 'connector' = 'kafka',
 'topic' = 'products_binlog',
 'properties.bootstrap.servers' = 'localhost:9092',
 'properties.group.id' = 'testGroup',
 'format' = 'canal-json'  -- using canal-json as the format
)
```

其中 DDL 中的表的字段和类型要和 mysql 中的字段及类型能匹配的上，接下来我们就可以写 flink sql 来查询我们定义的 topic_products了。

##### 2.1.1.2 debezium format

在国外，比较有名的类似 canal 的开源工具有 debezium，它的功能较 canal 更加强大一些，不仅仅支持 mysql。还支持其他的数据库的同步，比如 PostgreSQL、Oracle 等，目前 debezium 支持的序列化格式为 JSON 和 Apache Avro 。

debezium提供的格式如下：

```json
{
 "before": {
   "id": 111,
   "name": "scooter",
   "description": "Big 2-wheel scooter",
   "weight": 5.18
 },
 "after": {
   "id": 111,
   "name": "scooter",
   "description": "Big 2-wheel scooter",
   "weight": 5.15
 },
 "source": {...},
 "op": "u",
 "ts_ms": 1589362330904,
 "transaction": null
}
```

同样，使用 flink sql 来消费的时候，sql 和上面使用 canal 类似，只需要把 foramt 改成 debezium-json 即可。

##### 2.1.1.3 CanalJson 反序列化源码解析

接下来我们看下 flink 的源码中 canal-json 格式的实现。canal 格式作为一种 flink 的格式，而且是 source，所以也就是涉及到读取数据的时候进行反序列化，我们接下来就简单看看 CanalJson 的反序列化的实现。具体的实现类是 CanalJsonDeserializationSchema。

我们看下这个最核心的反序列化方法：

```java
@Override
public void deserialize(byte[] message, Collector<RowData> out) throws IOException {
  try {
    //使用json反序列化器将message反序列化成RowData
    RowData row = jsonDeserializer.deserialize(message);

    //获取type字段，用于下面的判断
    String type = row.getString(2).toString();
    if (OP_INSERT.equals(type)) {
      // 如果操作类型是insert，则data数组表示的是要插入的数据，则循环遍历data，然后添加一个标识INSERT，构造RowData对象，发送下游。
      ArrayData data = row.getArray(0);
      for (int i = 0; i < data.size(); i++) {
        RowData insert = data.getRow(i, fieldCount);
        insert.setRowKind(RowKind.INSERT);
        out.collect(insert);
      }
    } else if (OP_UPDATE.equals(type)) {
      // 如果是update操作，从data字段里获取更新后的数据、
      ArrayData data = row.getArray(0);
      // old字段获取更新之前的数据
      ArrayData old = row.getArray(1);
      for (int i = 0; i < data.size(); i++) {
        // the underlying JSON deserialization schema always produce GenericRowData.
        GenericRowData after = (GenericRowData) data.getRow(i, fieldCount);
        GenericRowData before = (GenericRowData) old.getRow(i, fieldCount);
        for (int f = 0; f < fieldCount; f++) {
          if (before.isNullAt(f)) {
            //如果old字段非空，则说明进行了数据的更新，如果old字段是null，则说明更新前后数据一样，这个时候把before的数据也设置成after的，也就是发送给下游的before和after数据一样。
            before.setField(f, after.getField(f));
          }
        }
        before.setRowKind(RowKind.UPDATE_BEFORE);
        after.setRowKind(RowKind.UPDATE_AFTER);
        //把更新前后的数据都发送下游
        out.collect(before);
        out.collect(after);
      }
    } else if (OP_DELETE.equals(type)) {
      // 如果是删除操作，data字段里包含将要被删除的数据，把这些数据组织起来发送给下游
      ArrayData data = row.getArray(0);
      for (int i = 0; i < data.size(); i++) {
        RowData insert = data.getRow(i, fieldCount);
        insert.setRowKind(RowKind.DELETE);
        out.collect(insert);
      }
    } else {
      if (!ignoreParseErrors) {
        throw new IOException(format(
          "Unknown \"type\" value \"%s\". The Canal JSON message is '%s'", type, new String(message)));
      }
    }
  } catch (Throwable t) {
    // a big try catch to protect the processing.
    if (!ignoreParseErrors) {
      throw new IOException(format(
        "Corrupt Canal JSON message '%s'.", new String(message)), t);
    }
  }
}
```

### 2.2 基于 Flink SQL CDC 同步方案

下面给大家带来 3 个关于 Flink SQL + CDC 在实际场景中使用较多的案例。在完成实验时候，你需要 Docker、MySQL、Elasticsearch 等组件，具体请参考每个案例参考文档。

#### 2.2.1 案例 

##### 1. Flink SQL CDC + JDBC Connector

这个案例通过订阅我们订单表（事实表）数据，通过 Debezium 将 MySQL Binlog 发送至 Kafka，通过维表 Join 和 ETL 操作把结果输出至下游的 PG 数据库。具体可以参考 Flink 公众号文章：《Flink JDBC Connector：Flink 与数据库集成最佳实践》案例进行实践操作。

<img src="https://ucc.alicdn.com/pic/developer-ecology/9fbb325b6cfc46d0b6ea3414fb0a473f.png" style="zoom:67%;" />

##### 2. CDC Streaming ETL

模拟电商公司的订单表和物流表，需要对订单数据进行统计分析，对于不同的信息需要进行关联后续形成订单的大宽表后，交给下游的业务方使用 ES 做数据分析，这个案例演示了如何只依赖 Flink 不依赖其他组件，借助 Flink 强大的计算能力实时把 Binlog 的数据流关联一次并同步至 ES 。

<img src="https://ucc.alicdn.com/pic/developer-ecology/fe46d5094ddc4f778aa81380ba4a4866.png" alt="10.png" style="zoom:67%;" />

例如如下的这段 Flink SQL 代码就能完成实时同步 MySQL 中 orders 表的全量+增量数据的目的。

```sql
CREATE TABLE orders (
  order_id INT,
  order_date TIMESTAMP(0),
  customer_name STRING,
  price DECIMAL(10, 5),
  product_id INT,
  order_status BOOLEAN
) WITH (
  'connector' = 'mysql-cdc',
  'hostname' = 'localhost',
  'port' = '3306',
  'username' = 'root',
  'password' = '123456',
  'database-name' = 'mydb',
  'table-name' = 'orders'
);

SELECT * FROM orders
```

为了让读者更好地上手和理解，我们还提供了 docker-compose 的测试环境，更详细的案例教程请参考下文的视频链接和文档链接。

> 视频链接：
> https://www.bilibili.com/video/BV1zt4y1D7kt
> 文档教程：
> https://github.com/ververica/flink-cdc-connectors/wiki/中文教程

##### 3. Streaming Changes to Kafka

下面案例就是对 GMV 进行天级别的全站统计。包含插入/更新/删除，只有付款的订单才能计算进入 GMV ，观察 GMV 值的变化。

![11.png](https://ucc.alicdn.com/pic/developer-ecology/708256232bc64896abc327a7cf0a01ae.png)

> 视频链接：
> https://www.bilibili.com/video/BV1zt4y1D7kt
> 文档教程：
> https://github.com/ververica/flink-cdc-connectors/wiki/中文教程

#### 2.2.2 Flink SQL CDC 的更多应用场景

Flink SQL CDC 不仅可以灵活地应用于实时数据同步场景中，还可以打通更多的场景提供给用户选择。

##### Flink 在数据同步场景中的灵活定位

· 如果你已经有 Debezium/Canal + Kafka 的采集层 (E)，可以使用 Flink 作为计算层 (T) 和传输层 (L)
· 也可以用 Flink 替代 Debezium/Canal ，由 Flink 直接同步变更数据到 Kafka，Flink 统一 ETL 流程
· 如果不需要 Kafka 数据缓存，可以由 Flink 直接同步变更数据到目的地，Flink 统一 ETL 流程

##### Flink SQL CDC : 打通更多场景

· 实时数据同步，数据备份，数据迁移，数仓构建
优势：丰富的上下游（E & L），强大的计算（T），易用的 API（SQL），流式计算低延迟
· 数据库之上的实时物化视图、流式数据分析
· 索引构建和实时维护
· 业务 cache 刷新
· 审计跟踪
· 微服务的解耦，读写分离
· 基于 CDC 的维表关联

下面介绍一下为何用 CDC 的维表关联会比基于查询的维表查询快。

**■ 基于查询的维表关联**

<img src="https://ucc.alicdn.com/pic/developer-ecology/48d3e43e60ae4c2cb6e810d26d24ac44.png" alt="12.png" style="zoom:67%;" />

目前维表查询的方式主要是通过 Join 的方式，数据从消息队列进来后通过向数据库发起 IO 的请求，由数据库把结果返回后合并再输出到下游，但是这个过程无可避免的产生了 IO 和网络通信的消耗，导致吞吐量无法进一步提升，就算使用一些缓存机制，但是因为缓存更新不及时可能会导致精确性也没那么高。

**■ 基于 CDC 的维表关联**

<img src="https://ucc.alicdn.com/pic/developer-ecology/095230f5273d4dd2b92468f64eced044.png" alt="13.png" style="zoom:67%;" />

我们可以通过 CDC 把维表的数据导入到维表 Join 的状态里面，在这个 State 里面因为它是一个分布式的 State ，里面保存了 Database 里面实时的数据库维表镜像，当消息队列数据过来时候无需再次查询远程的数据库了，直接查询本地磁盘的 State ，避免了 IO 操作，实现了低延迟、高吞吐，更精准。

Tips：目前此功能在 1.12 版本的规划中，具体进度请关注 FLIP-132 。

#### 2.2.3 mysql-cdc

目前 flink 支持两种内置的 connector，PostgreSQL 和 mysql，接下来我们以 mysql 为例简单讲讲。

在使用之前，我们需要引入相应的 pom，mysql 的 pom 如下：

```xml
<dependency>
  <groupId>com.alibaba.ververica</groupId>
  <!-- add the dependency matching your database -->
  <artifactId>flink-connector-mysql-cdc</artifactId>
  <version>1.1.0</version>
</dependency>
```

如果是 sql 客户端使用，需要下载 flink-sql-connector-mysql-cdc-1.1.0.jar 并且放到<FLINK_HOME>/lib/下面

连接 mysql 数据库的示例 sql 如下：

```sql
CREATE TABLE mysql_binlog (
 id INT NOT NULL,
 name STRING,
 description STRING,
 weight DECIMAL(10,3)
) WITH (
 'connector' = 'mysql-cdc',
 'hostname' = 'localhost',
 'port' = '3306',
 'username' = 'flinkuser',
 'password' = 'flinkpw',
 'database-name' = 'inventory',
 'table-name' = 'products'
)
```

如果订阅的是postgres数据库，我们需要把connector替换成postgres-cdc，DDL中表的schema和数据库一一对应。

更加详细的配置参见：https://github.com/ververica/flink-cdc-connectors/wiki/MySQL-CDC-Connector

## 三、mysql-cdc connector源码解析

接下来我们以 mysql-cdc 为例，看看源码层级是怎么实现的。既然作为一个 sql 的 connector，那么就首先会有一个对应的 TableFactory，然后在工厂类里面构造相应的 source，最后将消费下来的数据转成 flink 认识的 RowData格式，发送到下游。

我们按照这个思路来看看 flink cdc 源码的实现。

在 flink-connector-mysql-cdc module 中，找到其对应的工厂类：MySQLTableSourceFactory，进入createDynamicTableSource(Context context)方法，在这个方法里，使用从 ddl 中的属性里获取的 host、dbname 等信息构造了一个MySQLTableSource 类。

### 3.1 MySQLTableSource

在 `MySQLTableSource#getScanRuntimeProvider` 方法里，我们看到，首先构造了一个用于序列化的对象 `RowDataDebeziumDeserializeSchema`，这个对象主要是用于将 Debezium 获取的 `SourceRecord` 格式的数据转化为 flink 认识的`RowData` 对象。 我们看下 `RowDataDebeziumDeserializeSchem#deserialize` 方法，这里的操作主要就是先判断下进来的数据类型（insert 、update、delete），然后针对不同的类型（short、int等）分别进行转换，

最后我们看到用于 flink 用于获取数据库变更日志的 Source 函数是 `DebeziumSourceFunction`，且最终返回的类型是 `RowData`。

也就是说 flink 底层是采用了 Debezium 工具从 mysql、postgres 等数据库中获取的变更数据。

```java
	@SuppressWarnings("unchecked")
	@Override
	public ScanRuntimeProvider getScanRuntimeProvider(ScanContext scanContext) {
		RowType rowType = (RowType) physicalSchema.toRowDataType().getLogicalType();
		TypeInformation<RowData> typeInfo = (TypeInformation<RowData>) scanContext.createTypeInformation(physicalSchema.toRowDataType());
		DebeziumDeserializationSchema<RowData> deserializer = new RowDataDebeziumDeserializeSchema(
			rowType,
			typeInfo,
			((rowData, rowKind) -> {}),
			serverTimeZone);
		MySQLSource.Builder<RowData> builder = MySQLSource.<RowData>builder()
			.hostname(hostname)
			..........
		DebeziumSourceFunction<RowData> sourceFunction = builder.build();

		return SourceFunctionProvider.of(sourceFunction, false);
	}
```

### 3.2 DebeziumSourceFunction

我们接下来看看 `DebeziumSourceFunction` 类

```java
@PublicEvolving
public class DebeziumSourceFunction<T> extends RichSourceFunction<T> implements
		CheckpointedFunction,
		ResultTypeQueryable<T> {
		    .............
		}
```

我们看到 `DebeziumSourceFunction` 类继承了 `RichSourceFunction`，并且实现了 `CheckpointedFunction` 接口，也就是说这个类是flink 的一个 `SourceFunction`，会从源端（run方法）获取数据，发送给下游。此外这个类还实现了 `CheckpointedFunction` 接口，也就是会通过 checkpoint 的机制来保证 exactly once 语义。

接下来我们进入run方法，看看是如何获取数据库的变更数据的。

```java
	@Override
	public void run(SourceContext<T> sourceContext) throws Exception {
        ...........................
		// DO NOT include schema change, e.g. DDL
		properties.setProperty("include.schema.changes", "false");
         ...........................
        //将所有的属性信息打印出来，以便排查。
		// dump the properties
		String propsString = properties.entrySet().stream()
			.map(t -> "\t" + t.getKey().toString() + " = " + t.getValue().toString() + "\n")
			.collect(Collectors.joining());
		LOG.info("Debezium Properties:\n{}", propsString);
		
		//用于具体的处理数据的逻辑
		this.debeziumConsumer = new DebeziumChangeConsumer<>(
			sourceContext,
			deserializer,
			restoredOffsetState == null, // DB snapshot phase if restore state is null
			this::reportError);

		// create the engine with this configuration ...
		this.engine = DebeziumEngine.create(Connect.class)
			.using(properties)
			.notifying(debeziumConsumer)  // 数据发给上面的debeziumConsumer
			.using((success, message, error) -> {
				if (!success && error != null) {
					this.reportError(error);
				}
			})
			.build();

		if (!running) {
			return;
		}

		// run the engine asynchronously
		executor.execute(engine);

        //循环判断，当程序被打断，或者有错误的时候，打断engine，并且抛出异常
		// on a clean exit, wait for the runner thread
		try {
			while (running) {
				if (executor.awaitTermination(5, TimeUnit.SECONDS)) {
					break;
				}
				if (error != null) {
					running = false;
					shutdownEngine();
					// rethrow the error from Debezium consumer
					ExceptionUtils.rethrow(error);
				}
			}
		}
		catch (InterruptedException e) {
			// may be the result of a wake-up interruption after an exception.
			// we ignore this here and only restore the interruption state
			Thread.currentThread().interrupt();
		}
	}
```

在函数的开始，设置了很多的 properties，比如 include.schema.changes 设置为 false，也就是不包含表的DDL操作，表结构的变更是不捕获的。我们这里只关注数据的增删改。

接下来构造了一个 `DebeziumChangeConsumer` 对象，这个类实现了 `DebeziumEngine.ChangeConsumer` 接口，主要就是将获取到的一批数据进行一条条的加工处理。

接下来定一个 DebeziumEngine 对象，这个对象是真正用来干活的，它的底层使用了 kafka 的 connect-api 来进行获取数据，得到的是一个`org.apache.kafka.connect.source.SourceRecord` 对象。通过 notifying 方法将得到的数据交给上面定义的`DebeziumChangeConsumer` 来来覆盖缺省实现以进行复杂的操作。

接下来通过一个线程池 ExecutorService 来异步的启动这个 engine。

最后，做了一个循环判断，当程序被打断，或者有错误的时候，打断 engine，并且抛出异常。

总结一下，就是在 Flink 的 source 函数里，使用 Debezium 引擎获取对应的数据库变更数据（SourceRecord），经过一系列的反序列化操作，最终转成了 flink 中的 RowData 对象，发送给下游。

## 四、changelog format

### 4.1 使用场景

当我们从 mysql-cdc 获取数据库的变更数据，或者写了一个 group by 的查询的时候，这种结果数据都是不断变化的，我们如何将这些变化的数据发到只支持 append mode 的 kafka 队列呢?

于是 flink 提供了一种 changelog format，其实我们非常简单的理解为，flink 对进来的 RowData 数据进行了一层包装，然后加了一个数据的操作类型，包括以下几种 INSERT,DELETE, UPDATE_BEFORE,UPDATE_AFTER。这样当下游获取到这个数据的时候，就可以根据数据的类型来判断下如何对数据进行操作了。

比如我们的原始数据格式是这样的。

```json
{"day":"2020-06-18","gmv":100}
```

经过 changelog 格式的加工之后，成为了下面的格式：

```json
{"data":{"day":"2020-06-18","gmv":100},"op":"+I"}
```

也就是说 changelog format 对原生的格式进行了包装，添加了一个op字段，表示数据的操作类型，目前有以下几种：

- +I：插入操作。
- -U ：更新之前的数据内容：
- +U ：更新之后的数据内容。
- -D ：删除操作。

### 4.2 示例

使用的时候需要引入相应的 pom

```xml
<dependency>
  <groupId>com.alibaba.ververica</groupId>
  <artifactId>flink-format-changelog-json</artifactId>
  <version>1.1.0</version>
</dependency>
```

使用flink sql操作的方式如下：

```sql
CREATE TABLE kafka_gmv (
  day_str STRING,
  gmv DECIMAL(10, 5)
) WITH (
    'connector' = 'kafka',
    'topic' = 'kafka_gmv',
    'scan.startup.mode' = 'earliest-offset',
    'properties.bootstrap.servers' = 'localhost:9092',
    'format' = 'changelog-json'
);
```

我们定义了一个 format 为 changelog-json 的kafka connector，之后我们就可以对其进行写入和查询了。

完整的代码和配置请参考：https://github.com/ververica/flink-cdc-connectors/wiki/Changelog-JSON-Format

### 4.3 源码浅析

作为一种 flink 的 format ，我们主要看下其序列化和发序列化方法，changelog-json 使用了flink-json包进行json的处理。

#### 4.3.1 反序列化

反序列化用的是 `ChangelogJsonDeserializationSchema` 类，在其构造方法里，我们看到主要是构造了一个json的序列化器jsonDeserializer用于对数据进行处理。

```java
	public ChangelogJsonDeserializationSchema(
			RowType rowType,
			TypeInformation<RowData> resultTypeInfo,
			boolean ignoreParseErrors,
			TimestampFormat timestampFormatOption) {
		this.resultTypeInfo = resultTypeInfo;
		this.ignoreParseErrors = ignoreParseErrors;
		this.jsonDeserializer = new JsonRowDataDeserializationSchema(
			createJsonRowType(fromLogicalToDataType(rowType)),
			// the result type is never used, so it's fine to pass in Debezium's result type
			resultTypeInfo,
			false, // ignoreParseErrors already contains the functionality of failOnMissingField
			ignoreParseErrors,
			timestampFormatOption);
	}
```

其中 `createJsonRowType` 方法指定了 changelog 的 format 是一种 Row 类型的格式，我们看下代码：

```java
private static RowType createJsonRowType(DataType databaseSchema) {
		DataType payload = DataTypes.ROW(
			DataTypes.FIELD("data", databaseSchema),
			DataTypes.FIELD("op", DataTypes.STRING()));
		return (RowType) payload.getLogicalType();
	}
```

在这里，指定了这个 row 格式有两个字段，一个是 data，表示数据的内容，一个是 op，表示操作的类型。

最后看下最核心的 `ChangelogJsonDeserializationSchema#deserialize(byte[] bytes, Collector<RowData> out>)`

```java
	@Override
	public void deserialize(byte[] bytes, Collector<RowData> out) throws IOException {
		try {
			GenericRowData row = (GenericRowData) jsonDeserializer.deserialize(bytes);
			GenericRowData data = (GenericRowData) row.getField(0);
			String op = row.getString(1).toString();
			RowKind rowKind = parseRowKind(op);
			data.setRowKind(rowKind);
			out.collect(data);
		} catch (Throwable t) {
			// a big try catch to protect the processing.
			if (!ignoreParseErrors) {
				throw new IOException(format(
					"Corrupt Debezium JSON message '%s'.", new String(bytes)), t);
			}
		}
	}
```

使用 jsonDeserializer 对数据进行处理，然后对第二个字段op进行判断，添加对应的 RowKind。

#### 4.3.2 序列化

序列化的方法我们看下方法：`ChangelogJsonSerializationSchema#serialize`

```java
	@Override
	public byte[] serialize(RowData rowData) {
		reuse.setField(0, rowData);
		reuse.setField(1, stringifyRowKind(rowData.getRowKind()));
		return jsonSerializer.serialize(reuse);
	}
	
```

这块没有什么难度，就是将 flink 的 RowData 使用 jsonSerializer 序列化成字节数组。



## 五、未来规划

· FLIP-132 ：Temporal Table DDL（基于 CDC 的维表关联）
· Upsert 数据输出到 Kafka
· 更多的 CDC formats 支持（debezium-avro, OGG, Maxwell）
· 批模式支持处理 CDC 数据
· flink-cdc-connectors 支持更多数据库



## 六、Q&A

1. Debezium snapshot 的各种模式?

   [官网](https://debezium.io/documentation/reference/1.5/connectors/mysql.html)

   在 Debezium mysql connector 的配置中，用 `snapshot.mode` 配置项表示快照模式。在 Flink 中可以使用 `debezium.snapshot.mode` 来配置。它的值有以下几种情况:

   - initial (也是默认的模式)

     大多数情况下，MySQL 的 binlog 不再包含数据库的完整历史操作（为了节省磁盘空间，MySQL只会记录一定时间内或者一定大小的binlog）。slave(Debezium) 的 mysql connector 启动的时候，它会默认执行一次数据库初始的一致性快照任务。

   - when_needed

     允许 connector 在有必要的任何时候执行快照任务。这个和上面默认的`initial`快照相似

     唯一不同的是：如果 connector 重启并且 MySQL 不再有 connector 启动时读取的 binlog 位置，连接器将执行另一次快照任务而不是失败。这种模式也许是最自动化的，但是在遇到故障时（通常指连接器挂掉很长时间）存在执行额外快照的风险。

   - never

     保证了 connector 从不执行快照任务。当一个新的 connector 配置成这种模式的时候，它会从 binlog 的起始位置开始读。这不是默认的行为，因为这种模式的 connector （没有快照）要求 MySQL 的 binlog 包含连接器所监控的数据库的所有历史，通常 MySQL 实例不会这么配置，即不会保存所有历史的 binlog。特别地，binlog 必须包含连接器监控的每个表的至少 `CREATE TABLE` 语句。如果这个要求不能满足，connector 将不能合理解释 binlog 中的行级别事件的结构信息，这样connector 就会简单粗暴地跳过所有丢失表定义的事件。（connector 不能依赖表的当前 schema 信息，因为表可能会在binlog 中记录初始事件后被改变（alter），connector 没办法根据当前的 schema 合理解释 binlog 中的事件）

   - schema_only

     从 Debezium 0.3.4 版本开始，第四种快照模式 `schema_only` 出现了，它允许 connector 启动后从它当前所在 MySQL binlog 位置开始读。通过 `schema_only` 模式，connector 从当前 binlog 位置开始读，捕获当前表的 schema 信息，不读取任何数据信息，然后从当前位置开始继续读取binlog。这样导致改变事件流（change event stream）只包括快照开始后发生的改变事件。这种模式对于那些不需要知道数据库完整状态，只想知道 connector 启动后的改变的消费者很有用。

   - schema_only_recovery

     从 Debezium 0.7.2开始，第五种快照模式 `schema_only_recovery`，允许一个存在的 connector 去恢复中断的或者丢失的数据库历史 topic。它和 `schema_only` 模式很像，它也不读取任何数据信息，只捕获当前表 schema 信息。不同的地方在于：

     - 它只可以被用在已经存在的 connector 上，作为对 connector 配置的更新项。
     - 它从已经存在的 connector 上次提交的binlog位置开始读起，而不是 binlog 的当前位置。 

   

2. 重复 snapshot 的问题?

   每个用于读取 binlog 的 MySQL 数据库客户端都应具有唯一的ID，称为 server id。MySQL 服务器将使用此 ID 维护网络连接和 binlog 位置。如果不同的作业共享相同的 server id，则可能导致从错误的 binlog 位置进行读取。

   > 提示：默认情况下，启动 TaskManager 时，server id 是随机的。如果 TaskManager 失败，则再次启动时，它可能具有不同的 server id。但这不应该经常发生（作业异常不会重新启动 TaskManager），也不会对 MySQL 服务器造成太大影响。

   因此，建议为每个作业设置不同的 server id ，例如：

   - 通过 [SQL Hints](https://links.jianshu.com/go?to=https%3A%2F%2Fci.apache.org%2Fprojects%2Fflink%2Fflink-docs-release-1.11%2Fdev%2Ftable%2Fsql%2Fhints.html) **： SELECT \* FROM source_table /\*+ OPTIONS('server-id'='123456') \*/ ;**
   - 通过 Stream API 创建 source 时设置： `MySQLSource.builder()**.xxxxxx**.serverId(123456);`

   **重点**：Mysql 的 binlog 可以说是针对库级别，所以相同的 server id 去拉一个库里的不同表或者相同表可能会造成数据丢失。所以建议设置 server id。[传送门](http://apache-flink.147419.n8.nabble.com/Flink-CDC-packet-td8875.html)

