# Nebula Graph 查询语言（nGQL）

本文介绍 Nebula Graph 使用的查询语言 nGQL（Nebula Graph Query Language）。

## 一、什么是nGQL

nGQL 是 Nebula Graph 使用的的声明式图查询语言，支持灵活高效的[图模式](https://docs.nebula-graph.com.cn/2.0.1/3.ngql-guide/1.nGQL-overview/3.graph-patterns/)，而且 nGQL 是为开发和运维人员设计的类 SQL 查询语言，易于学习。

nGQL 是一个进行中的项目，会持续发布新特性和优化，因此可能会出现语法和实际操作不一致的问题，如果遇到此类问题，请提交[issue](https://github.com/vesoft-inc/nebula-graph/issues)通知 Nebula Graph 团队。Nebula Graph 2.0 及更新版本将支持 [openCypher 9](https://www.opencypher.org/resources)。

## 二、nGQL可以做什么

- 支持图遍历

- 支持模式匹配

- 支持聚合

- 支持修改图

- 支持访问控制

- 支持聚合查询

- 支持索引

- 支持大部分openCypher 9图查询语法（不支持修改和控制语法）

## 三、准备

### 3.1 导入示例数据 Basketballplayer

用户可以下载 Nebula Graph 示例数据[basketballplayer文件](https://docs.nebula-graph.io/2.0/basketballplayer-2.X.ngql)，然后使用 [Nebula Graph Console](https://docs.nebula-graph.com.cn/2.0.1/2.quick-start/3.connect-to-nebula-graph/)，使用选项`-f`执行脚本。

>1. Linux/Mac
>
>$ ./nebula-console -addr <ip> -port <port> -u <username> -p <password>
>[-t 120] [-e "nGQL_statement" | -f filename.nGQL]
>
>| `-h`           | 显示帮助菜单。                                               |
>| -------------- | ------------------------------------------------------------ |
>| `-addr`        | 设置要连接的graphd服务的IP地址。默认地址为127.0.0.1。        |
>| `-port`        | 设置要连接的graphd服务的端口。默认端口为9669。               |
>| `-u/-user`     | 设置Nebula Graph账号的用户名。未启用身份认证时，用户名可以填写任意字符。 |
>| `-p/-password` | 设置用户名对应的密码。未启用身份认证时，密码可以填写任意字符。 |
>| `-t/-timeout`  | 设置整数类型的连接超时时间。单位为秒，默认值为120。          |
>| `-e/-eval`     | 设置字符串类型的nGQL语句。连接成功后会执行一次该语句并返回结果，然后自动断开连接。 |
>| `-f/-file`     | 设置存储nGQL语句的文件的路径。连接成功后会执行该文件内的nGQL语句并返回结果，执行完毕后自动断开连接。 |

```shell
# sherlock @ mbp in ~/Downloads/nebula-data/basketballplayer [13:13:16]
$ nebula-console -addr 10.10.110.241 -port 9669 -u root -p nebula -f /Users/sherlock/Downloads/nebula-data/basketballplayer/basketballplayer-2.X.ngql
.....
(root@nebula) [basketballplayer]> insert edge serve(start_year,end_year) values "player149"->"team219":(2016, 2019);
Execution succeeded (time spent 638/1477 us)

Mon, 05 Jul 2021 13:13:15 CST

(root@nebula) [basketballplayer]> insert edge serve(start_year,end_year) values "player150"->"team213":(2018, 2019);
Execution succeeded (time spent 648/1558 us)

Mon, 05 Jul 2021 13:13:15 CST

Bye root!
Mon, 05 Jul 2021 13:13:15 CST
```

### 3.2 占位标识符和占位符值

Nebula Graph查询语言nGQL参照以下标准设计：

- ISO/IEC 10646

- ISO/IEC 39075

- ISO/IEC NP 39075 (Draft)

- OpenCypher 9

在模板代码中，任何非关键字、字面值或标点符号的标记都是占位符标识符或占位符值。

nGQL的符号说明如下。

| 符号 | 含义             |
| ---- | ---------------- |
| < >  | 语法元素的名称。 |
| ::=  | 定义元素的公式。 |
| [ ]  | 可选元素。       |
| { }  | 显示指定元素。   |
| \|   | 所有可选的元素。 |
| ...  | 可以重复多次。   |

例如创建点或边的nGQL语法：

```sql
CREATE {TAG | EDGE} {<tag_name> | <edge_type>}(<property_name> <data_type>
[, <property_name> <data_type> ...]);
```

示例语句：

```sql
nebula> CREATE TAG player(name string, age int);
```

## 四、基础操作语法 (CRUD)

### 4.1 图空间(SPACE)和Schema

一个 Nebula Graph 实例由一个或多个图空间(SPACE) 组成。每个图空间都是物理隔离的，用户可以在同一个实例中使用不同的图空间存储不同的数据集。

![](https://docs-cdn.nebula-graph.com.cn/docs-2.0/2.quick-start/nebula-graph-instance-and-graph-spaces.png)

为了在图空间(SPACE)中插入数据，需要为图数据库定义一个 Schema。Nebula Graph 的 Schema是由如下几部分组成。

| 组成部分            | 说明                                               |
| ------------------- | -------------------------------------------------- |
| 点（vertex）        | 表示现实世界中的实体。一个点可以有一个或多个标签。 |
| 标签（tag）         | 点的类型，定义了一组描述点类型的属性。             |
| 边（edge）          | 表示两个点之间**有方向**的关系。                   |
| 边类型（edge type） | 边的类型，定义了一组描述边类型的属性。             |

### 4.2 数据模型

Nebula Graph 的数据模型。数据模型是一种组织数据并说明它们如何相互关联的模型。

#### 4.2.1 数据结构

Nebula Graph 数据模型使用 6 种基本的数据结构：

- 图空间(space）

  图空间用于隔离不同团队或者项目的数据。不同图空间的数据是相互隔离的，可以指定不同的存储副本数、权限、分片等。

- 点（vertex）

  点用来保存实体对象，特点如下：

  - 点是用点标识符（`VID`）标识的。`VID`在同一图空间中唯一。VID 是一个 int64/fixed_string(N)。
  - 点必须有至少一个标签（Tag)，也可以有多个标签。

- 边（edge）

  边是用来连接点的，表示两个点之间的关系或行为，特点如下：

  - 两点之间可以有多条边。
  - 边是有方向的，不存在无向边。
  - 四元组 `<起点VID、边类型（edge type）、边排序值(rank)、终点VID>` 用于唯一标识一条边。边没有EID。
  - 一条边有且仅有一个边类型。
  - 一条边有且仅有一个 rank。其为int64, 默认为0。

- 标签（tag）

  标签由一组事先预定义的属性构成。

- 边类型（edge type）

  边类型由一组事先预定义的属性构成。

- 属性（properties）

  属性是指以键值对（key-value pair）形式存储的信息。

#### 4.2.2 有向属性图

Nebula Graph 使用有向属性图模型，指点和边构成的图，这些边是有方向的，点和边都可以有属性。

下表为篮球运动员数据集的结构示例，包括两种类型的点（**player**、**team**）和两种类型的边（**serve**、**follow**）。

| 类型      | 名称       | 属性名（数据类型）              | 说明                                                         |
| --------- | ---------- | ------------------------------- | ------------------------------------------------------------ |
| tag       | **player** | name(string), age(int)          | 表示球员。                                                   |
| tag       | **team**   | name(string)                    | 表示球队。                                                   |
| edge type | **serve**  | start_year(int), end_year (int) | 表示球员的行为。 该行为将球员和球队联系起来，方向是从球员到球队。 |
| edge type | **follow** | degree(int)                     | 表示球员的行为。 该行为将两个球员联系起来，方向是从一个球员到另一个球员。 |

### 4.3 CRUD 前准备

接下来将使用下图的数据集演示基础操作的语法。

<img src="https://docs-cdn.nebula-graph.com.cn/docs-2.0/2.quick-start/dataset-for-crud.png" style="zoom:75%;" />

#### 4.3.1 检查Nebula Graph集群的机器状态

```shell
(root@nebula) [(none)]> show hosts;
+-------------+------+----------+--------------+-------------------------------------------------------------------------+-------------------------------------------------------------------------+
| Host        | Port | Status   | Leader count | Leader distribution                                                     | Partition distribution                                                  |
+-------------+------+----------+--------------+-------------------------------------------------------------------------+-------------------------------------------------------------------------+
| "127.0.0.1" | 9779 | "ONLINE" | 4            | "basketballplayer:1, lineage_demo:1, sink_flink:1, test_iris_lineage:1" | "basketballplayer:1, lineage_demo:1, sink_flink:1, test_iris_lineage:1" |
+-------------+------+----------+--------------+-------------------------------------------------------------------------+-------------------------------------------------------------------------+
| "Total"     |      |          | 4            | "basketballplayer:1, lineage_demo:1, sink_flink:1, test_iris_lineage:1" | "basketballplayer:1, lineage_demo:1, sink_flink:1, test_iris_lineage:1" |
+-------------+------+----------+--------------+-------------------------------------------------------------------------+-------------------------------------------------------------------------+
Got 2 rows (time spent 1339/4607 us)
```

在返回结果中，查看**Status**列，可以看到所有Storage服务都在线。

#### 4.3.2 异步实现创建和修改

Nebula Graph 中执行如下创建和修改操作，是异步实现的，需要在下一个心跳周期才同步数据。

- `CREATE SPACE`
- `CREATE TAG`
- `CREATE EDGE`
- `ALTER TAG`
- `ALTER EDGE`
- `CREATE TAG INDEX`
- `CREATE EDGE INDEX`

> Note
>
> 默认心跳周期是10秒。修改心跳周期参数 `heartbeat_interval_secs`，请参见[配置简介](https://docs.nebula-graph.com.cn/2.0.1/5.configurations-and-logs/1.configurations/1.configurations/)。

为确保数据同步，后续操作能顺利进行，可采取以下方法之一：

- 执行 `SHOW` 或 `DESCRIBE` 命令检查相应对象的状态，确保创建或修改已完成。如果没有完成，请等待几秒重试。

- 等待 2 个心跳周期（20秒）。

### 4.4 CRUD

#### 4.4.1 创建和选择图空间

##### 4.4.1.1 nGQL语法

- 创建图空间

  ```sql
  CREATE SPACE [IF NOT EXISTS] <graph_space_name>
      [(partition_num = <partition_number>, 
      replica_factor = <replica_number>, 
      vid_type = {FIXED_STRING(<N>) | INT64})];
  ```

  | 参数           | 说明                                                         |
  | -------------- | ------------------------------------------------------------ |
  | partition_num  | 指定图空间的分片数量。建议设置为5倍的集群硬盘数量。例如集群中有3个硬盘，建议设置15个分片。 |
  | replica_factor | 指定每个分片的副本数量。建议在生产环境中设置为3，在测试环境中设置为1。由于需要进行基于quorum的选举，副本数量必须是**奇数**。 |
  | vid_type       | 指定点ID的数据类型。可选值为`FIXED_STRING(<N>)`和`INT64`。`FIXED_STRING(<N>)`表示数据类型为字符串，最大长度为`N`，超出长度会报错；`INT64`表示数据类型为整数。默认值为`FIXED_STRING(8)`。 |

- 列出创建成功的图空间

  ```sql
  nebula> SHOW SPACES;
  ```

- 选择数据库

  ```sql
  USE <graph_space_name>;
  ```

##### 4.4.1.2 示例

1. 执行如下语句创建名为`basketballplayer`的图空间。

   ```sql
   (root@nebula) [(none)]> CREATE SPACE IF NOT EXISTS basketballplayer ( \
                        ->   partition_num = 1, \
                        ->   replica_factor = 1, \
                        ->   charset = utf8, \
                        ->   collate = utf8_bin, \
                        ->   vid_type = INT64 \
                        -> );
   Execution succeeded (time spent 927/2190 us)
   ```

2. 执行命令 `SHOW HOSTS` 检查分片的分布情况，确保平衡分布。

   ```sh
   (root@nebula) [(none)]> SHOW HOSTS;
   +-------------+------+----------+--------------+-------------------------------------------------------------------------+-------------------------------------------------------------------------+
   | Host        | Port | Status   | Leader count | Leader distribution                                                     | Partition distribution                                                  |
   +-------------+------+----------+--------------+-------------------------------------------------------------------------+-------------------------------------------------------------------------+
   | "127.0.0.1" | 9779 | "ONLINE" | 4            | "basketballplayer:1, lineage_demo:1, sink_flink:1, test_iris_lineage:1" | "basketballplayer:1, lineage_demo:1, sink_flink:1, test_iris_lineage:1" |
   +-------------+------+----------+--------------+-------------------------------------------------------------------------+-------------------------------------------------------------------------+
   | "Total"     |      |          | 4            | "basketballplayer:1, lineage_demo:1, sink_flink:1, test_iris_lineage:1" | "basketballplayer:1, lineage_demo:1, sink_flink:1, test_iris_lineage:1" |
   +-------------+------+----------+--------------+-------------------------------------------------------------------------+-------------------------------------------------------------------------+
   Got 2 rows (time spent 1051/2336 us)
   ```

   如果 **Leader distribution** 分布不均匀，请执行命令 `BALANCE LEADER` 重新分配。更多信息，请参见[Storage负载均衡](https://docs.nebula-graph.com.cn/2.0.1/8.service-tuning/load-balance/)。

3. 选择图空间 `basketballplayer`。

   ```shell
   (root@nebula) [(none)]> USE basketballplayer;
   Execution succeeded (time spent 935/2156 us)
   ```

   用户可以执行命令 `SHOW SPACES` 查看创建的图空间。

   ```shell
   (root@nebula) [basketballplayer]> SHOW SPACES;
   +---------------------+
   | Name                |
   +---------------------+
   | "basketballplayer"  |
   +---------------------+
   | "lineage_demo"      |
   +---------------------+
   | "sink_flink"        |
   +---------------------+
   | "test_iris_lineage" |
   +---------------------+
   Got 4 rows (time spent 913/2176 us)
   ```

#### 4.4.2 创建标签和边类型

##### 4.4.2.1 nGQL语法

```sql
CREATE {TAG | EDGE} {<tag_name> | <edge_type>}(<property_name> <data_type>
[, <property_name> <data_type> ...]);
```

##### 4.4.2.2 示例

创建标签 `player` 和 `team`，以及边类型 `follow` 和 `serve`。说明如下表。

| 名称   | 类型      | 属性                             |
| ------ | --------- | -------------------------------- |
| player | Tag       | name (string), age (int)         |
| team   | Tag       | name (string)                    |
| follow | Edge type | degree (int)                     |
| serve  | Edge type | start_year (int), end_year (int) |

```shell
(root@nebula) [basketballplayer]> CREATE TAG IF NOT EXISTS player(name string, age int);
Execution succeeded (time spent 936/2441 us)

Wed, 24 Feb 2021 03:47:01 EST

(root@nebula) [basketballplayer]> CREATE TAG IF NOT EXISTS team(name string);
Execution succeeded (time spent 935/2348 us)

Wed, 24 Feb 2021 03:47:59 EST

(root@nebula) [basketballplayer]> CREATE EDGE IF NOT EXISTS follow(degree int);
Execution succeeded (time spent 921/2196 us)

Wed, 24 Feb 2021 03:48:07 EST

(root@nebula) [basketballplayer]> CREATE EDGE IF NOT EXISTS serve(start_year int, end_year int);
Execution succeeded (time spent 898/4544 us)

Wed, 24 Feb 2021 03:48:16 EST
```

#### 4.4.3 插入点和边

用户可以使用 `INSERT` 语句，基于现有的标签插入点，或者基于现有的边类型插入边。

##### 4.4.3.1 nGQL语法

- 插入点

  ```sql
  INSERT VERTEX <tag_name> (<property_name>[, <property_name>...])
  [, <tag_name> (<property_name>[, <property_name>...]), ...]
  {VALUES | VALUE} <vid>: (<property_value>[, <property_value>...])
  [, <vid>: (<property_value>[, <property_value>...];
  ```

  `VID` 是Vertex ID的缩写，`VID` 在一个图空间中是唯一的。

- 插入边

  ```sql
  INSERT EDGE <edge_type> (<property_name>[, <property_name>...])
  {VALUES | VALUE} <src_vid> -> <dst_vid>[@<rank>] : (<property_value>[, <property_value>...])
  [, <src_vid> -> <dst_vid>[@<rank>] : (<property_name>[, <property_name>...]), ...];
  ```

##### 4.4.3.2 示例

- 插入代表球员和球队的点。

  ```sh
  nebula> INSERT VERTEX player(name, age) VALUES "player100":("Tim Duncan", 42);
  Execution succeeded (time spent 28196/30896 us)
  
  Wed, 24 Feb 2021 03:55:08 EST
  
  nebula> INSERT VERTEX player(name, age) VALUES "player101":("Tony Parker", 36);
  Execution succeeded (time spent 2708/3834 us)
  
  Wed, 24 Feb 2021 03:55:20 EST
  
  nebula> INSERT VERTEX player(name, age) VALUES "player102":("LaMarcus Aldridge", 33);
  Execution succeeded (time spent 1945/3294 us)
  
  Wed, 24 Feb 2021 03:55:32 EST
  
  nebula> INSERT VERTEX team(name) VALUES "team200":("Warriors"), "team201":("Nuggets");
  Execution succeeded (time spent 2269/3310 us)
  
  Wed, 24 Feb 2021 03:55:47 EST
  ```

- 插入代表球员和球队之间关系的边。

  ```sh
  nebula> INSERT EDGE follow(degree) VALUES "player100" -> "player101":(95);
  Execution succeeded (time spent 3362/4542 us)
  
  Wed, 24 Feb 2021 03:57:36 EST
  
  nebula> INSERT EDGE follow(degree) VALUES "player100" -> "player102":(90);
  Execution succeeded (time spent 2974/4274 us)
  
  Wed, 24 Feb 2021 03:57:44 EST
  
  nebula> INSERT EDGE follow(degree) VALUES "player102" -> "player101":(75);
  Execution succeeded (time spent 1891/3096 us)
  
  Wed, 24 Feb 2021 03:57:52 EST
  
  nebula> INSERT EDGE serve(start_year, end_year) VALUES "player100" -> "team200":(1997, 2016), "player101" -> "team201":(1999,  2018);
  Execution succeeded (time spent 6064/7104 us)
  
  Wed, 24 Feb 2021 03:58:01 EST
  ```

#### 4.4.4 查询数据

常用查询语言

- [GO](https://docs.nebula-graph.com.cn/2.0.1/3.ngql-guide/7.general-query-statements/3.go/) 语句可以根据指定的条件遍历数据库。`GO` 语句从一个或多个点开始，沿着一条或多条边遍历，返回 `YIELD` 子句中指定的信息。

- [FETCH](https://docs.nebula-graph.com.cn/2.0.1/3.ngql-guide/7.general-query-statements/4.fetch/) 语句可以获得点或边的属性。

- [LOOKUP](https://docs.nebula-graph.com.cn/2.0.1/3.ngql-guide/7.general-query-statements/5.lookup/) 语句是基于[索引](https://docs.nebula-graph.com.cn/2.0.1/2.quick-start/4.nebula-graph-crud/#_14)的，和`WHERE`子句一起使用，查找符合特定条件的数据。

- [MATCH](https://docs.nebula-graph.com.cn/2.0.1/3.ngql-guide/7.general-query-statements/2.match/)语句是查询图数据最常用的，但是它依赖[索引](https://docs.nebula-graph.com.cn/2.0.1/2.quick-start/4.nebula-graph-crud/#_14)去匹配Nebula Graph中的数据模型。

##### 4.4.4.1 nGQL语法

- `GO`

  ```sql
  GO [[<M> TO] <N> STEPS ] FROM <vertex_list>
  OVER <edge_type_list> [REVERSELY] [BIDIRECT]
  [WHERE <expression> [AND | OR expression ...])]
  YIELD [DISTINCT] <return_list>;
  ```

- `FETCH`

  - 查询标签属性

    ```sql
    FETCH PROP ON {<tag_name> | <tag_name_list> | *} <vid_list>
    [YIELD [DISTINCT] <return_list>];
    ```

  - 查询边属性

    ```sql
    FETCH PROP ON <edge_type> <src_vid> -> <dst_vid>[@<rank>]
    [, <src_vid> -> <dst_vid> ...]
    [YIELD [DISTINCT] <return_list>];
    ```

- `LOOKUP`

  ```sql
  LOOKUP ON {<tag_name> | <edge_type>} 
  WHERE <expression> [AND expression ...])]
  [YIELD <return_list>];
  ```

- `MATCH`

  ```sql
  MATCH <pattern> [<WHERE clause>] RETURN <output>;
  ```

##### 4.4.4.2 GO语句示例

- 从 VID 为 `player121` 的球员开始，沿着边 `follow/serve` 找到连接的球员。

  ```sql
  (root@nebula) [basketballplayer]> GO FROM "player121" OVER follow;
  +-------------+
  | follow._dst |
  +-------------+
  | "player116" |
  +-------------+
  | "player128" |
  +-------------+
  | "player129" |
  +-------------+
  Got 3 rows (time spent 1148/2383 us)
  
  (root@nebula) [basketballplayer]> GO FROM "player121" OVER serve;
  +------------+
  | serve._dst |
  +------------+
  | "team202"  |
  +------------+
  | "team207"  |
  +------------+
  | "team215"  |
  +------------+
  Got 3 rows (time spent 1213/11469 us)
  ```

- 从 VID 为 `player121` 的球员开始，沿着边 `follow` 查找年龄大于或等于 30 岁的球员，并返回他们的姓名和年龄，同时重命名对应的列。

  ```sql
  (root@nebula) [basketballplayer]> GO FROM "player121" OVER follow WHERE $$.player.age >= 30 \
        												 -> YIELD $$.player.name AS teammate, $$.player.age AS age;
  +-------------------+-----+
  | teammate          | age |
  +-------------------+-----+
  | "LeBron James"    | 34  |
  +-------------------+-----+
  | "Carmelo Anthony" | 34  |
  +-------------------+-----+
  | "Dwyane Wade"     | 37  |
  +-------------------+-----+
  Got 3 rows (time spent 1992/3374 us)
  ```

  | 子句/符号 | 说明                           |
  | --------- | ------------------------------ |
  | `YIELD`   | 指定该查询需要返回的值或结果。 |
  | `$$`      | 表示边的终点。                 |
  | `\`       | 表示换行继续输入。             |

- 从 VID 为 `player121` 的球员开始，沿着边 `follow` 查找连接的球员，然后检索这些球员的球队。为了合并这两个查询请求，可以使用管道符或临时变量。

  - 使用管道符

    ```sql
    (root@nebula) [basketballplayer]> GO FROM "player121" OVER follow YIELD follow._dst AS id | \
                                   -> GO FROM $-.id OVER serve YIELD $$.team.name AS Team, \
                                   -> $^.player.name AS Player;
    +-------------+-------------------+
    | Team        | Player            |
    +-------------+-------------------+
    | "Lakers"    | "LeBron James"    |
    +-------------+-------------------+
    | "Heat"      | "LeBron James"    |
    +-------------+-------------------+
    | "Cavaliers" | "LeBron James"    |
    +-------------+-------------------+
    | "Cavaliers" | "LeBron James"    |
    +-------------+-------------------+
    | "Nuggets"   | "Carmelo Anthony" |
    +-------------+-------------------+
    | "Rockets"   | "Carmelo Anthony" |
    +-------------+-------------------+
    | "Thunders"  | "Carmelo Anthony" |
    +-------------+-------------------+
    | "Knicks"    | "Carmelo Anthony" |
    +-------------+-------------------+
    | "Cavaliers" | "Dwyane Wade"     |
    +-------------+-------------------+
    | "Bulls"     | "Dwyane Wade"     |
    +-------------+-------------------+
    | "Heat"      | "Dwyane Wade"     |
    +-------------+-------------------+
    | "Heat"      | "Dwyane Wade"     |
    +-------------+-------------------+
    Got 12 rows (time spent 3026/4619 us)
    ```

    | 子句/符号 | 说明                                                       |
    | --------- | ---------------------------------------------------------- |
    | `$^`      | 表示边的起点。                                             |
    | `|`       | 组合多个查询的管道符，将前一个查询的结果集用于后一个查询。 |
    | `$-`      | 表示管道符前面的查询输出的结果集。                         |

  - 使用临时变量

    >Note
    >
    >当复合语句作为一个整体提交给服务器时，其中的临时变量会在语句结束时被释放。

    ```sql
    (root@nebula) [basketballplayer]> $var = GO FROM "player121" OVER follow YIELD follow._dst AS id; \
                                   -> GO FROM $var.id OVER serve YIELD $$.team.name AS Team, \
                                   -> $^.player.name AS Player;
    +-------------+-------------------+
    | Team        | Player            |
    +-------------+-------------------+
    | "Lakers"    | "LeBron James"    |
    +-------------+-------------------+
    | "Heat"      | "LeBron James"    |
    +-------------+-------------------+
    | "Cavaliers" | "LeBron James"    |
    +-------------+-------------------+
    | "Cavaliers" | "LeBron James"    |
    +-------------+-------------------+
    | "Nuggets"   | "Carmelo Anthony" |
    +-------------+-------------------+
    | "Rockets"   | "Carmelo Anthony" |
    +-------------+-------------------+
    | "Thunders"  | "Carmelo Anthony" |
    +-------------+-------------------+
    | "Knicks"    | "Carmelo Anthony" |
    +-------------+-------------------+
    | "Cavaliers" | "Dwyane Wade"     |
    +-------------+-------------------+
    | "Bulls"     | "Dwyane Wade"     |
    +-------------+-------------------+
    | "Heat"      | "Dwyane Wade"     |
    +-------------+-------------------+
    | "Heat"      | "Dwyane Wade"     |
    +-------------+-------------------+
    Got 12 rows (time spent 3041/4482 us)
    ```

##### 4.4.4.3 FETCH语句示例

查询VID为`player100`的球员的属性。

```sql
(root@nebula) [basketballplayer]> FETCH PROP ON player "player121";
+----------------------------------------------------+
| vertices_                                          |
+----------------------------------------------------+
| ("player121" :player{age: 33, name: "Chris Paul"}) |
+----------------------------------------------------+
Got 1 rows (time spent 1188/4553 us)
```

> Note
>
> `LOOKUP` 和 `MATCH` 的示例在下文的[索引](https://docs.nebula-graph.com.cn/2.0.1/2.quick-start/4.nebula-graph-crud/#_14)部分查看。

