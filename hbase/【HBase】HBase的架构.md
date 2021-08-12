## 【HBase】HBase的架构

### 一、HBase体系结构

HBase 的宏观架构如下图:

![HBase宏观架构](/Users/sherlock/Desktop/notes/allPics/HBase/HBase宏观架构.png)

从上图可以看出, HBase 集群是由一个 Master (也可以把两个 Master 做成高可用的冷热主节点)和多个 RegionServer 组成.

1. ***HBase客户端***

   HBase 客户端 (Client) 提供了 Shell 命令行接口, 原生 Java API编程接口, Thrift/REST API接口以及 MapReduce编程接口. 其中Thrift/REST API接口主要用于支持非 Java 的上层业务需求, MR 接口主要用于批量数据的读取和导入

   HBase 客户端访问数据行之前, 首先需要通过访问 Zookeeper 获取存储元数据表 ***hbase:meta*** 的 RegionServer 的地址, 之后才会将请求发送到目标 RegionServer. 同时将这些元数据会被缓存在客户端本地, 如果集群 RegionServer 发生宕机或执行了负载均衡导致数据分片的位置的发生迁移, 客户端需要重新请求最新的元数据并缓存于本地.

2. ***Zookeeper***

   在 HBase 系统中, Zookeeper 扮演者非常重要的角色

   - ***实现 Master 的高可用*** : 一旦 Active Master 发生宕机, Zookeeper 会检测到该宕机事件, 并通过一定机制选举出新的 Master 以保证系统的正常运转.
   - ***管理系统的核心元数据*** : 比如管理当前系统中正常工作的 RegionServer 的集合, 保存系统元数据表 hbase: meta 表所在 RegionServer 的地址信息等.
   - ***参与 RegionServer 的宕机恢复*** : Zookeeper 通过心跳可以感知到 RegionServer 是否宕机, 并在宕机后通知 Master 进行宕机处理
   - ***实现分布式表锁*** : HBase 中对一张表的各种管理操作 (比如 alter 操作) 需要先加表锁, 防止其他用户对同一张表进行管理操作, 造成表状态不一致. 和其他 RDBMS 表不同, HBase 中的表通常都是分布式存储, ZK 可以通过特定机制实现分布式表锁

3. ***Master (HMaster)***

   主要负责 HBase 系统的各种管理工作

   - 处理用户的各种管理请求, 包括建表、修改表、权限操作、切分表、合并数据分片以及 Compaction等.
   - 管理集群中所有的 RegionServer, 包括 RegionServer 中的 region 的负载均衡、RegionServer 的宕机恢复以及 region 的迁移等.
   - 清理过期的日志及文件, Master 会每隔一段时间检查 HDFS 中 HLog 是否过期、 HFile 是否已被删除, 并在过期之后将其删除

4. ***RegionServer***

   主要负责响应客户的 IO 请求, 是 HBase 中最核心的模块, 由 WAL (HLog)、BlockCache 以及多个 region 组成

   - ***WAL (HLog)*** : HLog 在 HBase 中主要有两个核心作用

     1. 用于实现数据的高可靠型, HBase 数据随机写入时, 并非直接写入 HFile 数据文件, 而是先写入缓存, 再异步刷新落盘. 为了防止缓存数据丢失, 数据写入缓存前必须先顺序写入 HLog, 这样即使数据丢失, 仍然可以通过 HLog 日志恢复;
     2. 用户实现 HBase 集群间的主从复制, 通过回放主机群推送过来的 HLog 实现主从复制

   - ***BolckCache*** : HBase 系统中的读缓存. 客户端从磁盘读取数据之后通常会将数据缓存到系统内存中, 后续访问同一行数据可以直接从内存中获取而不需要访问磁盘, 对于带有大量热点业务的请求来说, 缓存机制极大地提升了性能.

     BlockCache 缓存的对象是一系列 block 块, 一个 block 默认为 64KB, 由物理上相邻的多个 KV 数据组成, BlockCache 同时利用了 `空间局部性` 和 `时间局部性` 

     前者表示最近读取的 KV 数据很可能与当前读取的数据在地址上是相邻的, 缓存单位是 Block 而不是单个 KV 就可以实现空间局部性;

     后者表示一个 KV 数据正在被访问, 那么它近期还可能再次被访问. 

     当前 BlockCache 主要有两种实现 --- LRUBlockCache 和 BucketCache, 前期实现相对简单, 而后者在对 GC 优化方面有明显的提升.

   - ***Region***

     数据表的一个分片, 当数据表大小超过一定阈值就会"水平切分", 分裂为两个 Region. Region 是集群负载均衡的基本单位. 通常一张表的 region 会分布在整个集群的多台 RegionServer 上, 一个 RegionServer 上会管理多个 Region, 当然, 这些 Region 一般来自于不同的数据表.

     

   一个 Region 由一个或多个 Store 组成, Store的数量取决于当前 region 对应的数据表中的列簇的个数, 多少个列簇就会有多少个 Store. HBase 中每个列簇的数据都集中存放一起形成一个存储单元 Store, 因此也建议将具有相同的 IO 特性的数据设置在同一个列簇中

   每一个 Store 由一个 MemStore 和多个 HFile 构成. MemStore 简称为写缓存, 用户写入数据时首先会写到 MemStore, 当 MemStore 写满之后 (缓存数据超过阈值, 默认 128 MB) 系统会异步将数据 flush 成一个 HFile 文件. 显然, 随着数据不断写入, HFile 文件会越来越多, 当 HFile 文件数超过一定阈值后系统将会执行 Compact 操作, 将这些小文件通过一定策略合并成一个或多个大文件.

5. ***HDFS***

   HBase 底层还是依赖于 HDFS 组件存储实际数据, 包括用户数据文件、HLog 日志文件等最终都会写入 HDFS 落盘. HDFS 是 Hadoop 生态圈最成熟的组件之一, 数据默认三副本存储策略有效保证数据的高可用性. HBase 背部封装一个名为 DFSClient 的 HDFS 客户端组件, 负责对 HDFS 的实际数据进行读写访问

#### 1.1 RegionServer

一个 RegionServer 内部是多个 Region 的集合, 上面也提到了, Region 相当于是一张表的一个分区

![RegionServer](/Users/sherlock/Desktop/notes/allPics/HBase/RegionServer.png)

从上图中可以看出一个 RegionServer 中包含有

- WAL(HLog)

  预写日志, WAL 是 Write Ahead Log 的缩写. 从名字可以看出它的作用就是: 预先写入. 当操作到达 region 时, HBase 就先把操作写入到 WAL 里面去. HBase 会先把数据存放到基于内存的 MemStore里, 等数据达到一定数量级(或者约定好的刷写时间)才会刷写 (flush) 最终存储到 HFile 里, 而如果在这个过程中服务器宕机或者断电了, 那么数据就丢失了. WAL 是一个保险机制, 数据在写入到 MemStore之前, 会先被写入到 WAL, 这样当出现故障就可以从 WAL 中恢复数据

- Region

  前面说过, region 中存储表的一部分数据, 相当于数据的分片, 每一个 region 都有起始 rowkey 和结束 rowkey, 代表了它所存储的 row 的范围.

### 二、HBase 系统特性

1. HBase 的优点

2. HBase 的缺点

   任何一个系统都不可能完美, HBase 也一样, HBase 不能适用于所有的业务场景, 如:

   - HBase 本身不支持复杂的聚合运算 (如 join, GroupBy 等), 如果业务中有复杂的聚合运算, 可以在 HBase 之上假设 Phoenix 组件或者 Spark/Flink 组件, 前者主要用于小规模的 OLTP 场景, 后者应用于大规模聚合的 OLTP 场景
   - HBase 本身没有实现二级索引功能, 所以不支持二级索引功能查找. 好在针对 HBase 实现的第三方二级索引方案非常丰富, 比如目前比较普遍使用的 Phoenix 提供的二级索引功能
   - HBase 原生不支持全局跨行事务, 只支持单行事务模型. 同样可以使用 Phoenix 提供的全局事务模型组件来弥补 HBase 这一缺陷



##### 1.1.1 Region

![region](/Users/sherlock/Desktop/notes/allPics/HBase/region.png)

一个 Region 内部包含有一个或多个 Store实例, ***一个 Store 对应于一个列簇的数量***, 如果一个表有两个列簇, 那么这张表的 region中就有两个 Store, 而在下面 Store 的解析图中可以看出 ***Store 内部有 MemStore 和 HFile 这两个组成部分***.

##### 1.1.2 WAL