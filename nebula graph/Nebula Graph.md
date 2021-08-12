# Nebula Graph

## 一、什么是图数据库

图数据库是专门存储庞大的图形网络并从中检索信息的数据库。它可以将图中的数据高效存储为点（vertex）和边（edge），还可以将属性（property）附加到点和边上。

![What is a graph database](https://docs-cdn.nebula-graph.com.cn/docs-2.0/1.introduction/what-is-a-graph-database.png)

图数据库适合存储大多数从现实抽象出的数据类型。世界上几乎所有领域的事物都有内在联系，像关系型数据库这样的建模系统会提取实体之间的关系，并将关系单独存储到表和列中，而实体的类型和属性存储在其他列甚至其他表中，这使得数据管理费时费力。

Nebula Graph作为一个典型的原生图数据库，允许将丰富的关系存储为边，边的类型和属性可以直接附加到边上。

## 二、数据结构

Nebula Graph 数据模型使用6种基本的数据结构：

- 图空间(space）

  图空间用于隔离不同团队或者项目的数据。不同图空间的数据是相互隔离的，可以指定不同的存储副本数、权限、分片等。

- 点（vertex）

  点用来保存实体对象，特点如下：

  - 点是用点标识符（`VID`）标识的。`VID`在同一图空间中唯一。VID 是一个 int64, 或者 fixed_string(N)。
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

### 2.1 有向属性图

Nebula Graph使用有向属性图模型，指点和边构成的图，这些边是有方向的，点和边都可以有属性。

下表为篮球运动员数据集的结构示例，包括两种类型的点（**player**、**team**）和两种类型的边（**serve**、**follow**）。

| 类型      | 名称       | 属性名（数据类型）                  | 说明                                                         |
| --------- | ---------- | ----------------------------------- | ------------------------------------------------------------ |
| tag       | **player** | name （string） age （int）         | 表示球员。                                                   |
| tag       | **team**   | name （string）                     | 表示球队。                                                   |
| edge type | **serve**  | start_year （int） end_year （int） | 表示球员的行为。 该行为将球员和球队联系起来，方向是从球员到球队。 |
| edge type | **follow** | degree（int）                       | 表示球员的行为。 该行为将两个球员联系起来，方向是从一个球员到另一个球员。 |

## 三、架构总览

Nebula Graph由三种服务构成：Graph服务、Meta服务和Storage服务。

每个服务都有可执行的二进制文件和对应进程，用户可以使用这些二进制文件在一个或多个计算机上部署Nebula Graph集群。

下图展示了Nebula Graph集群的经典架构。

![Nebula Graph architecture](https://docs-cdn.nebula-graph.com.cn/docs-2.0/1.introduction/2.nebula-graph-architecture/nebula-graph-architecture-1.png)

### 3.1 Meta服务

在Nebula Graph架构中，Meta服务是由nebula-metad进程提供的，负责数据管理，例如Schema操作、集群管理和用户权限管理等。

Meta服务的详细说明，请参见[Meta服务](https://docs.nebula-graph.com.cn/2.0.1/1.introduction/3.nebula-graph-architecture/2.meta-service/)。

### 3.2 Graph服务和Storage服务

Nebula Graph 采用计算存储分离架构。Graph 服务负责处理计算请求，Storage 服务负责存储数据。它们由不同的进程提供，Graph 服务是由 nebula-graphd 进程提供，Storage 服务是由 nebula-storaged 进程提供。计算存储分离架构的优势如下：

- 易扩展

  分布式架构保证了Graph服务和Storage服务的灵活性，方便扩容和缩容。

- 高可用

  如果提供Graph服务的服务器有一部分出现故障，其余服务器可以继续为客户端提供服务，而且Storage服务存储的数据不会丢失。服务恢复速度较快，甚至能做到用户无感知。

- 节约成本

  计算存储分离架构能够提高资源利用率，而且可根据业务需求灵活控制成本。如果使用[Nebula Graph Cloud](https://cloud.nebula-graph.com.cn/)，可以进一步节约前期成本。

- 更多可能性

  基于分离架构的特性，Graph服务将可以在更多类型的存储引擎上单独运行，Storage服务也可以为多种目的计算引擎提供服务。

