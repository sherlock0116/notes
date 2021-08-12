# 【Hive】HiveHook 和 Metastore 监听器介绍

元数据管理是数据仓库的核心，它不仅定义了数据仓库有什么，还指明了数据仓库中数据的内容和位置，刻画了数据的提取和转换规则，存储了与数据仓库主题有关的各种商业信息。本文主要介绍 Hive Hook 和 MetaStore Listener，使用这些功能可以进行自动的元数据管理。通过本文你可以了解到：

- **元数据管理**
- **Hive Hooks 和 Metastore Listeners**
- **Hive Hooks基本使用**
- **Metastore Listeners基本使用**

## 一、元数据管理

### 1.1 元数据定义

按照传统的定义，元数据（ Metadata  ）是关于数据的数据。元数据打通了源数据、数据仓库、数据应用，记录了数据从产生到消费的全过程。元数据主要记录数据仓库中模型的定义、各层级间的映射关系、监控数据仓库的数据状态及ETL  的任务运行状态。在数据仓库系统中，元数据可以帮助数据仓库管理员和开发人员非常方便地找到他们所关心的数据，用于指导其进行数据管理和开发工作，提高工作效率。将元数据按用途的不同分为两类：技术元数据（ Technical Metadata)和业务元数据（ Business Metadata  ）。技术元数据是存储关于数据仓库系统技术细节的数据，是用于开发和管理数据仓库使用的数据。

### 1.2 元数据分类

#### 1.2.1 技术元数据

- 分布式计算系统存储元数据

如Hive表、列、分区等信息。记录了表的表名。分区信息、责任人信息、文件大小、表类型，以及列的字段名、字段类型、字段备注、是否是分区字段等信息。

- 分布式计算系统运行元数据

  类似于Hive 的Job 日志，包括作业类型、实例名称、输入输出、SQL 、运行参数、执行时间等。

- 任务调度元数据

  任务的依赖类型、依赖关系等，以及不同类型调度任务的运行日志等。

#### 1.2.2 业务元数据

业务元数据从业务角度描述了数据仓库中的数据，它提供了介于使用者和实际系统之间的语义层，使得不懂计算机技术的业务人员也能够“  读懂”数据仓库中的数据。常见的业务元数据有：如维度及属性、业务过程、指标等的规范化定义，用于更好地管理和使用数据；数据应用元数据，如数据报表、数据产品等的配置和运行元数据。

### 1.3 元数据应用

数据的真正价值在于数据驱动决策，通过数据指导运营。通过数据驱动的方法，我们能够判断趋势，从而展开有效行动，帮助自己发现问题，推动创新或解决方案的产生。这就是数据化运营。同样，对于元数据，可以用于指导数据相关人员进行日常工作，实现数据化“运营”。比如对于数据使用者，可以通过元数据让其快速找到所需要的数据；对于ETL 工程师，可以通过元数据指导其进行模型设计、任务优化和任务下线等各种日常ETL  工作；对于运维工程师，可以通过元数据指导其进行整个集群的存储、计算和系统优化等运维工作。

## 二、Hive Hooks 和 Metastore Listeners

### 2.1 Hive Hooks

关于数据治理和元数据管理框架，业界有许多开源的系统，比如 **Apache Atlas**，这些开源的软件可以在复杂的场景下满足元数据管理的需求。其实 **Apache Atlas** 对于 Hive 的元数据管理，使用的是 Hive 的 Hooks。需要进行如下配置：

```xml
<property>
    <name>hive.exec.post.hooks</name>
    <value>org.apache.atlas.hive.hook.HiveHook<value/>
</property>
```

通过 Hook 监听 Hive 的各种事件，比如创建表，修改表等，然后按照特定的格式把收集的数据推送到 Kafka，最后消费元数据并存储。

#### 2.1.1 Hive Hooks 分类

**那么，究竟什么是Hooks呢？**

Hooks 是一种事件和消息机制， 可以将事件绑定在内部 Hive 的执行流程中，而无需重新编译 Hive。Hook 提供了扩展和继承外部组件的方式。根据不同的 Hook 类型，可以在不同的阶段运行。关于 Hooks 的类型，主要分为以下几种：

1. **hive.exec.pre.hooks**

   从名称可以看出，在执行引擎执行查询之前被调用。这个需要在 Hive 对查询计划进行过优化之后才可以使用。使用该Hooks需要实现接口：**org.apache.hadoop.hive.ql.hooks.ExecuteWithHookContext**，具体在**hive-site.xml**中的配置如下：

   ```xml
   <property>
       <name>hive.exec.pre.hooks</name>
       <value>实现类的全限定名<value/>
   </property>
   ```

   

2. **hive.exec.post.hooks**

   在执行计划执行结束结果返回给用户之前被调用。使用时需要实现接口：**org.apache.hadoop.hive.ql.hooks.ExecuteWithHookContext**，具体在**hive-site.xml**中的配置如下：

   ```xml
   <property>
       <name>hive.exec.post.hooks</name>
       <value>实现类的全限定名<value/>
   </property>
   ```

   

3. **hive.exec.failure.hooks**

   在执行计划失败之后被调用。使用时需要实现接口：**org.apache.hadoop.hive.ql.hooks.ExecuteWithHookContext**, 具体在**hive-site.xml**中的配置如下：

   ```xml
   <property>
       <name>hive.exec.failure.hooks</name>
       <value>实现类的全限定名<value/>
   </property>
   ```

   

4. **hive.metastore.init.hooks**

   HMSHandler初始化是被调用。使用时需要实现接口：**org.apache.hadoop.hive.metastore.MetaStoreInitListener**，具体在**hive-site.xml**中的配置如下：

   ```xml
   <property>
       <name>hive.metastore.init.hooks</name>
       <value>实现类的全限定名<value/>
   </property>
   ```

   

5. **hive.exec.driver.run.hooks**

   在 Driver.run 开始或结束时运行，使用时需要实现接口：**org.apache.hadoop.hive.ql.HiveDriverRunHook**，具体在**hive-site.xml**中的配置如下：

   ```xml
   <property>
       <name>hive.exec.driver.run.hooks</name>
       <value>实现类的全限定名<value/>
   </property>
   ```

   

6. **hive.semantic.analyzer.hook**

   Hive 对查询语句进行语义分析的时候调用。使用时需要集成抽象类：**org.apache.hadoop.hive.ql.parse.AbstractSemanticAnalyzerHook**，具体在**hive-site.xml**中的配置如下：

   ```xml
   <property>
       <name>hive.semantic.analyzer.hook</name>
       <value>实现类的全限定名<value/>
   </property>
   ```

   

#### 2.1.2 Hive Hooks 的优缺点

- 优点
  - 可以很方便地在各种查询阶段嵌入或者运行自定义的代码
  - 可以被用作更新元数据
- 缺点
  - 当使用 Hooks 时，获取到的元数据通常需要进一步解析，否则很难理解
  - 会影响查询的过程

> 对于 Hive Hooks，本文将给出 **hive.exec.post.hook** 的使用案例，该 Hooks 会在查询执行之后，返回结果之前运行。

### 2.2 Metastore Listeners

所谓 Metastore Listeners，指的是对 Hive metastore 的监听。用户可以自定义一些代码，用来使用对元数据的监听。

当我们看 **HiveMetaStore** 这个类的源码时，会发现：

在创建 HiveMetaStore 的 init() 方法中, 同时创建了三种 Listener, 分别为

- MetaStorePreEventListener
- MetaStoreEventListener
- MetaStoreEndFunctionListener

这些 Listener 用于对每一步事件的监听

```java
public class HiveMetaStore extends ThriftHiveMetastore {
  // ...省略代码
  public static class HMSHandler extends FacebookBase implements
    IHMSHandler {
    // ...省略代码
    public void init() throws MetaException {
      // ...省略代码
      // 获取MetaStorePreEventListener
      preListeners = MetaStoreUtils.getMetaStoreListeners(MetaStorePreEventListener.class,
                                                          hiveConf,
                                                          hiveConf.getVar(HiveConf.ConfVars.METASTORE_PRE_EVENT_LISTENERS));
      // 获取MetaStoreEventListener
      listeners = MetaStoreUtils.getMetaStoreListeners(MetaStoreEventListener.class,
                                                       hiveConf,
                                                       hiveConf.getVar(HiveConf.ConfVars.METASTORE_EVENT_LISTENERS));
      listeners.add(new SessionPropertiesListener(hiveConf));
      // 获取MetaStoreEndFunctionListener
      endFunctionListeners = MetaStoreUtils.getMetaStoreListeners(
        MetaStoreEndFunctionListener.class, 
        hiveConf,
        hiveConf.getVar(HiveConf.ConfVars.METASTORE_END_FUNCTION_LISTENERS));
      // ...省略代码
    }
  }
}
```

#### 2.2.1 Metastore Listeners 分类

1. **hive.metastore.pre.event.listeners**

   需要扩展此抽象类，以提供在metastore上发生特定事件之前需要执行的操作实现。在metastore上发生事件之前，将调用这些方法。

   使用时需要继承抽象类：**org.apache.hadoop.hive.metastore.MetaStorePreEventListener**，在**Hive-site.xml**中的配置为：

   ```xml
   <property>
   	<name>hive.metastore.pre.event.listeners</name>
   	<value>实现类的全限定名</value> 
   </property>
   ```

   

2. **hive.metastore.event.listeners**

   需要扩展此抽象类，以提供在metastore上发生特定事件时需要执行的操作实现。每当Metastore上发生事件时，就会调用这些方法。

   使用时需要继承抽象类：**org.apache.hadoop.hive.metastore.MetaStoreEventListener**，在**Hive-site.xml**中的配置为：

   ```xml
   <property>
   	<name>hive.metastore.event.listeners</name>
   	<value>实现类的全限定名</value> 
   </property>
   ```

   

3. **hive.metastore.end.function.listeners**

   每当函数结束时，将调用这些方法。

   使用时需要继承抽象类：**org.apache.hadoop.hive.metastore.MetaStoreEndFunctionListener** ，在**Hive-site.xml**中的配置为：

   ```xml
   <property>
       <name>hive.metastore.end.function.listeners</name>
       <value>实现类的全限定名</value> 
   </property>
   ```

   

#### 2.2.2 Metastore Listeners 优缺点

- 优点
  - 元数据已经被解析好了，很容易理解
  - 不影响查询的过程，是只读的
- 缺点
  - 不灵活，仅仅能够访问属于当前事件的对象

> 对于metastore listener，本文会给出 **MetaStoreEventListener** 的使用案例，具体会实现两个方法：onCreateTable 和 onAlterTable

## 三、Hive Hooks 基本使用

具体实现代码如下：

```java
public class CustomPostHook implements ExecuteWithHookContext {
  private static final Logger LOGGER = LoggerFactory.getLogger(CustomPostHook.class);
  // 存储Hive的SQL操作类型
  private static final HashSet<String> OPERATION_NAMES = new HashSet<>();

  // HiveOperation是一个枚举类，封装了Hive的SQL操作类型
  // 监控SQL操作类型
  static {
    // 建表
    OPERATION_NAMES.add(HiveOperation.CREATETABLE.getOperationName());
    // 修改数据库属性
    OPERATION_NAMES.add(HiveOperation.ALTERDATABASE.getOperationName());
    // 修改数据库属主
    OPERATION_NAMES.add(HiveOperation.ALTERDATABASE_OWNER.getOperationName());
    // 修改表属性,添加列
    OPERATION_NAMES.add(HiveOperation.ALTERTABLE_ADDCOLS.getOperationName());
    // 修改表属性,表存储路径
    OPERATION_NAMES.add(HiveOperation.ALTERTABLE_LOCATION.getOperationName());
    // 修改表属性
    OPERATION_NAMES.add(HiveOperation.ALTERTABLE_PROPERTIES.getOperationName());
    // 表重命名
    OPERATION_NAMES.add(HiveOperation.ALTERTABLE_RENAME.getOperationName());
    // 列重命名
    OPERATION_NAMES.add(HiveOperation.ALTERTABLE_RENAMECOL.getOperationName());
    // 更新列,先删除当前的列,然后加入新的列
    OPERATION_NAMES.add(HiveOperation.ALTERTABLE_REPLACECOLS.getOperationName());
    // 创建数据库
    OPERATION_NAMES.add(HiveOperation.CREATEDATABASE.getOperationName());
    // 删除数据库
    OPERATION_NAMES.add(HiveOperation.DROPDATABASE.getOperationName());
    // 删除表
    OPERATION_NAMES.add(HiveOperation.DROPTABLE.getOperationName());
  }

  @Override
  public void run(HookContext hookContext) throws Exception {
    assert (hookContext.getHookType() == HookType.POST_EXEC_HOOK);
    // 执行计划
    QueryPlan plan = hookContext.getQueryPlan();
    // 操作名称
    String operationName = plan.getOperationName();
    logWithHeader("执行的SQL语句: " + plan.getQueryString());
    logWithHeader("操作名称: " + operationName);
    if (OPERATION_NAMES.contains(operationName) && !plan.isExplain()) {
      logWithHeader("监控SQL操作");

      Set<ReadEntity> inputs = hookContext.getInputs();
      Set<WriteEntity> outputs = hookContext.getOutputs();

      for (Entity entity : inputs) {
        logWithHeader("Hook metadata输入值: " + toJson(entity));
      }

      for (Entity entity : outputs) {
        logWithHeader("Hook metadata输出值: " + toJson(entity));
      }

    } else {
      logWithHeader("不在监控范围，忽略该hook!");
    }

  }

  private static String toJson(Entity entity) throws Exception {
    ObjectMapper mapper = new ObjectMapper();
    //  entity的类型
    // 主要包括：
    // DATABASE, TABLE, PARTITION, DUMMYPARTITION, DFS_DIR, LOCAL_DIR, FUNCTION
    switch (entity.getType()) {
      case DATABASE:
        Database db = entity.getDatabase();
        return mapper.writeValueAsString(db);
      case TABLE:
        return mapper.writeValueAsString(entity.getTable().getTTable());
    }
    return null;
  }

  /**
     * 日志格式
     *
     * @param obj
     */
  private void logWithHeader(Object obj) {
    LOGGER.info("[CustomPostHook][Thread: " + Thread.currentThread().getName() + "] | " + obj);
  }

}
```

### 3.1 使用过程解释

首先将上述代码编译成 jar 包，放在 $HIVE_HOME/lib 目录下，或者使用在 Hive 的客户端中执行添加 jar 包的命令：

```sh
0: jdbc:hive2://localhost:10001> add jar /Users/sherlock/devTools/opt/hive-2.3.7/lib/iris-hive-hook-1.0-SNAPSHOT.jar;
```

接着配置 Hive-site.xml 文件，为了方便，我们直接使用客户端命令进行配置：       

```shell
0: jdbc:hive2://localhost:10001> set hive.exec.post.hooks=com.homedo.iris.hive.hook.HiveHook;
```

#### 3.1.1 DQL 操作

上面的代码中我们对一些操作进行了监控，当监控到这些操作时会触发一些自定义的代码(比如输出日志)。当我们在Hive的beeline客户端中输入下面命令时：     

```shell
0: jdbc:hive2://localhost:10001> show databases;
+----------------+--+
| database_name  |
+----------------+--+
| default        |
| sherlock       |
| ubt            |
+----------------+--+
3 rows selected (1.891 seconds)
0: jdbc:hive2://localhost:10001> show tables;
+-------------+--+
|  tab_name   |
+-------------+--+
| test_order  |
+-------------+--+
1 row selected (0.135 seconds)
0: jdbc:hive2://localhost:10001> select * from test_order limit 1;
+--------------------+--------------------+------------------------+------------------------+------------------------+----------------------+--+
| test_order.ord_id  | test_order.ord_no  | test_order.creat_date  | test_order.creat_time  | test_order.time_stamp  | test_order.log_time  |
+--------------------+--------------------+------------------------+------------------------+------------------------+----------------------+--+
+--------------------+--------------------+------------------------+------------------------+------------------------+----------------------+--+
No rows selected (3.136 seconds)
```

查看 ${HIVE_HOME}/hive.log

```shell
HiveServer2-Background-Pool: Thread-44] | ===> SQL: use sherlock
2021-06-06T15:13:31,908  INFO [HiveServer2-Background-Pool: Thread-44] hook.HiveHook: ==> [Custom Post HiveHook][Thread: HiveServer2-Background-Pool: Thread-44] | ===> OPERATION: SWITCHDATABASE
2021-06-06T15:13:31,908  INFO [HiveServer2-Background-Pool: Thread-44] hook.HiveHook: ==> [Custom Post HiveHook][Thread: HiveServer2-Background-Pool: Thread-44] | 不在监控范围，忽略该 hook!
......
HiveServer2-Background-Pool: Thread-47] | ===> SQL: show tables
2021-06-06T15:13:36,816  INFO [HiveServer2-Background-Pool: Thread-47] hook.HiveHook: ==> [Custom Post HiveHook][Thread: HiveServer2-Background-Pool: Thread-47] | ===> OPERATION: SHOWTABLES
2021-06-06T15:13:36,816  INFO [HiveServer2-Background-Pool: Thread-47] hook.HiveHook: ==> [Custom Post HiveHook][Thread: HiveServer2-Background-Pool: Thread-47] | 不在监控范围，忽略该 hook!
......
HiveServer2-Background-Pool: Thread-54] | ===> SQL: select * from test_order limit 1
2021-06-06T15:14:00,063  INFO [HiveServer2-Background-Pool: Thread-54] hook.HiveHook: ==> [Custom Post HiveHook][Thread: HiveServer2-Background-Pool: Thread-54] | ===> OPERATION: QUERY
2021-06-06T15:14:00,063  INFO [HiveServer2-Background-Pool: Thread-54] hook.HiveHook: ==> [Custom Post HiveHook][Thread: HiveServer2-Background-Pool: Thread-54] | ===> 监控 SQL 操作
2021-06-06T15:14:00,360  INFO [HiveServer2-Background-Pool: Thread-54] hook.HiveHook: ==> [Custom Post HiveHook][Thread: HiveServer2-Background-Pool: Thread-54] | 	 ===> Hook metadata 输入值: {"tableName":"test_order","dbName":"sherlock","owner":"sherlock","createTime":1616665274,"lastAccessTime":0,"retention":0,"sd":{"cols":[{"name":"ord_id","type":"string","comment":null,"setName":true,"setType":true,"setComment":false},{"name":"ord_no","type":"string","comment":null,"setName":true,"setType":true,"setComment":false},{"name":"creat_date","type":"string","comment":null,"setName":true,"setType":true,"setComment":false},{"name":"creat_time","type":"string","comment":null,"setName":true,"setType":true,"setComment":false},{"name":"time_stamp","type":"string","comment":null,"setName":true,"setType":true,"setComment":false}],"location":"hdfs://localhost:9000/user/hive/warehouse/sherlock.db/test_order","inputFormat":"org.apache.hadoop.mapred.TextInputFormat","outputFormat":"org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat","compressed":false,"numBuckets":-1,"serdeInfo":{"name":null,"serializationLib":"org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe","parameters":{"serialization.format":"\u0001","line.delim":"\n","field.delim":"\u0001"},"setName":false,"setParameters":true,"parametersSize":3,"setSerializationLib":true},"bucketCols":[],"sortCols":[],"parameters":{},"skewedInfo":{"skewedColNames":[],"skewedColValues":[],"skewedColValueLocationMaps":{},"setSkewedColNames":true,"setSkewedColValues":true,"setSkewedColValueLocationMaps":true,"skewedColNamesSize":0,"skewedColNamesIterator":[],"skewedColValuesSize":0,"skewedColValuesIterator":[],"skewedColValueLocationMapsSize":0},"storedAsSubDirectories":false,"colsSize":5,"colsIterator":[{"name":"ord_id","type":"string","comment":null,"setName":true,"setType":true,"setComment":false},{"name":"ord_no","type":"string","comment":null,"setName":true,"setType":true,"setComment":false},{"name":"creat_date","type":"string","comment":null,"setName":true,"setType":true,"setComment":false},{"name":"creat_time","type":"string","comment":null,"setName":true,"setType":true,"setComment":false},{"name":"time_stamp","type":"string","comment":null,"setName":true,"setType":true,"setComment":false}],"setCompressed":true,"setNumBuckets":true,"bucketColsSize":0,"bucketColsIterator":[],"sortColsSize":0,"sortColsIterator":[],"setStoredAsSubDirectories":true,"setCols":true,"setLocation":true,"setInputFormat":true,"setOutputFormat":true,"setSerdeInfo":true,"setBucketCols":true,"setSortCols":true,"setSkewedInfo":true,"setParameters":true,"parametersSize":0},"partitionKeys":[{"name":"log_time","type":"string","comment":null,"setName":true,"setType":true,"setComment":false}],"parameters":{"transient_lastDdlTime":"1616665274","comment":"for test"},"viewOriginalText":null,"viewExpandedText":null,"tableType":"MANAGED_TABLE","privileges":null,"temporary":false,"rewriteEnabled":false,"partitionKeysSize":1,"setOwner":true,"setPartitionKeys":true,"setViewOriginalText":false,"setViewExpandedText":false,"setTableType":true,"setRetention":true,"partitionKeysIterator":[{"name":"log_time","type":"string","comment":null,"setName":true,"setType":true,"setComment":false}],"setTemporary":false,"setRewriteEnabled":true,"setDbName":true,"setCreateTime":true,"setTableName":true,"setSd":true,"setParameters":true,"setPrivileges":false,"setLastAccessTime":true,"parametersSize":2}
2021-06-06T15:14:00,360  INFO [HiveServer2-Background-Pool: Thread-54] hook.HiveHook: ==> [Custom Post HiveHook][Thread: HiveServer2-Background-Pool: Thread-54] | 	 ===> Hook metadata输出值: null
```

这其中 metatdata input 输入值信息为:

```json
{
    "tableName":"test_order",
    "dbName":"sherlock",
    "owner":"sherlock",
    "createTime":1616665274,
    "lastAccessTime":0,
    "retention":0,
    "sd":{
        "cols":[
            {
                "name":"ord_id",
                "type":"string",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            },
            {
                "name":"ord_no",
                "type":"string",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            },
            {
                "name":"creat_date",
                "type":"string",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            },
            {
                "name":"creat_time",
                "type":"string",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            },
            {
                "name":"time_stamp",
                "type":"string",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            }
        ],
        "location":"hdfs://localhost:9000/user/hive/warehouse/sherlock.db/test_order",
        "inputFormat":"org.apache.hadoop.mapred.TextInputFormat",
        "outputFormat":"org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat",
        "compressed":false,
        "numBuckets":-1,
        "serdeInfo":{
            "name":null,
            "serializationLib":"org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe",
            "parameters":{
                "serialization.format":"\u0001",
                "line.delim":"
",
                "field.delim":"\u0001"
            },
            "setName":false,
            "setParameters":true,
            "parametersSize":3,
            "setSerializationLib":true
        },
        "bucketCols":[

        ],
        "sortCols":[

        ],
        "parameters":{

        },
        "skewedInfo":{
            "skewedColNames":[

            ],
            "skewedColValues":[

            ],
            "skewedColValueLocationMaps":{

            },
            "setSkewedColNames":true,
            "setSkewedColValues":true,
            "setSkewedColValueLocationMaps":true,
            "skewedColNamesSize":0,
            "skewedColNamesIterator":[

            ],
            "skewedColValuesSize":0,
            "skewedColValuesIterator":[

            ],
            "skewedColValueLocationMapsSize":0
        },
        "storedAsSubDirectories":false,
        "colsSize":5,
        "colsIterator":[
            {
                "name":"ord_id",
                "type":"string",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            },
            {
                "name":"ord_no",
                "type":"string",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            },
            {
                "name":"creat_date",
                "type":"string",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            },
            {
                "name":"creat_time",
                "type":"string",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            },
            {
                "name":"time_stamp",
                "type":"string",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            }
        ],
        "setCompressed":true,
        "setNumBuckets":true,
        "bucketColsSize":0,
        "bucketColsIterator":[

        ],
        "sortColsSize":0,
        "sortColsIterator":[

        ],
        "setStoredAsSubDirectories":true,
        "setCols":true,
        "setLocation":true,
        "setInputFormat":true,
        "setOutputFormat":true,
        "setSerdeInfo":true,
        "setBucketCols":true,
        "setSortCols":true,
        "setSkewedInfo":true,
        "setParameters":true,
        "parametersSize":0
    },
    "partitionKeys":[
        {
            "name":"log_time",
            "type":"string",
            "comment":null,
            "setName":true,
            "setType":true,
            "setComment":false
        }
    ],
    "parameters":{
        "transient_lastDdlTime":"1616665274",
        "comment":"for test"
    },
    "viewOriginalText":null,
    "viewExpandedText":null,
    "tableType":"MANAGED_TABLE",
    "privileges":null,
    "temporary":false,
    "rewriteEnabled":false,
    "partitionKeysSize":1,
    "setOwner":true,
    "setPartitionKeys":true,
    "setViewOriginalText":false,
    "setViewExpandedText":false,
    "setTableType":true,
    "setRetention":true,
    "partitionKeysIterator":[
        {
            "name":"log_time",
            "type":"string",
            "comment":null,
            "setName":true,
            "setType":true,
            "setComment":false
        }
    ],
    "setTemporary":false,
    "setRewriteEnabled":true,
    "setDbName":true,
    "setCreateTime":true,
    "setTableName":true,
    "setSd":true,
    "setParameters":true,
    "setPrivileges":false,
    "setLastAccessTime":true,
    "parametersSize":2
}
```

因为本地使用 HiveHook 主要是为了抓取数仓晚上跑批的 Hql

```sql
0: jdbc:hive2://localhost:10001> INSERT INTO t2 SELECT * FROM t1;
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
No rows affected (46.288 seconds)
0: jdbc:hive2://localhost:10001> SELECT * FROM t2;
+--------+----------+------------+--+
| t2.id  | t2.name  | t2.gender  |
+--------+----------+------------+--+
| 1      | jack     | male       |
| 2      | tom      | male       |
| 3      | lily     | female     |
| 4      | nico     | female     |
+--------+----------+------------+--+
```

对应日志:

```
2021-06-07T13:15:05,223  INFO [HiveServer2-Background-Pool: Thread-1452] hook.HiveHook: ==> [Custom Post HiveHook][Thread: HiveServer2-Background-Pool: Thread-1452] | ===> SQL: INSERT INTO t2 SELECT * FROM t1
2021-06-07T13:15:05,223  INFO [HiveServer2-Background-Pool: Thread-1452] hook.HiveHook: ==> [Custom Post HiveHook][Thread: HiveServer2-Background-Pool: Thread-1452] | ===> OPERATION: QUERY
2021-06-07T13:15:05,223  INFO [HiveServer2-Background-Pool: Thread-1452] hook.HiveHook: ==> [Custom Post HiveHook][Thread: HiveServer2-Background-Pool: Thread-1452] | ===> 监控 SQL 操作
2021-06-07T13:15:05,248  INFO [HiveServer2-Background-Pool: Thread-1452] hook.HiveHook: ==> [Custom Post HiveHook][Thread: HiveServer2-Background-Pool: Thread-1452] | 	 ===> Hook metadata 输入值: {"tableName":"t1","dbName":"sherlock","owner":"sherlock","createTime":1623042427,"lastAccessTime":0,"retention":0,"sd":{"cols":[{"name":"id","type":"int","comment":null,"setName":true,"setType":true,"setComment":false},{"name":"name","type":"string","comment":null,"setName":true,"setType":true,"setComment":false},{"name":"gender","type":"string","comment":null,"setName":true,"setType":true,"setComment":false}],"location":"hdfs://localhost:9000/user/hive/warehouse/sherlock.db/t1","inputFormat":"org.apache.hadoop.mapred.TextInputFormat","outputFormat":"org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat","compressed":false,"numBuckets":-1,"serdeInfo":{"name":null,"serializationLib":"org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe","parameters":{"serialization.format":",","field.delim":","},"setName":false,"setParameters":true,"parametersSize":2,"setSerializationLib":true},"bucketCols":[],"sortCols":[],"parameters":{},"skewedInfo":{"skewedColNames":[],"skewedColValues":[],"skewedColValueLocationMaps":{},"setSkewedColNames":true,"setSkewedColValues":true,"setSkewedColValueLocationMaps":true,"skewedColNamesSize":0,"skewedColNamesIterator":[],"skewedColValuesSize":0,"skewedColValuesIterator":[],"skewedColValueLocationMapsSize":0},"storedAsSubDirectories":false,"colsSize":3,"colsIterator":[{"name":"id","type":"int","comment":null,"setName":true,"setType":true,"setComment":false},{"name":"name","type":"string","comment":null,"setName":true,"setType":true,"setComment":false},{"name":"gender","type":"string","comment":null,"setName":true,"setType":true,"setComment":false}],"setCompressed":true,"setNumBuckets":true,"bucketColsSize":0,"bucketColsIterator":[],"sortColsSize":0,"sortColsIterator":[],"setStoredAsSubDirectories":true,"setCols":true,"setLocation":true,"setInputFormat":true,"setOutputFormat":true,"setSerdeInfo":true,"setBucketCols":true,"setSortCols":true,"setSkewedInfo":true,"setParameters":true,"parametersSize":0},"partitionKeys":[],"parameters":{"transient_lastDdlTime":"1623042628","totalSize":"50","numRows":"0","rawDataSize":"0","numFiles":"1"},"viewOriginalText":null,"viewExpandedText":null,"tableType":"MANAGED_TABLE","privileges":null,"temporary":false,"rewriteEnabled":false,"partitionKeysSize":0,"setOwner":true,"setPartitionKeys":true,"setViewOriginalText":false,"setViewExpandedText":false,"setTableType":true,"setRetention":true,"partitionKeysIterator":[],"setTemporary":false,"setRewriteEnabled":true,"setDbName":true,"setCreateTime":true,"setTableName":true,"setSd":true,"setParameters":true,"setPrivileges":false,"setLastAccessTime":true,"parametersSize":5}
2021-06-07T13:15:05,260  INFO [HiveServer2-Background-Pool: Thread-1452] hook.HiveHook: ==> [Custom Post HiveHook][Thread: HiveServer2-Background-Pool: Thread-1452] | 	 ===> Hook metadata输出值: {"tableName":"t2","dbName":"sherlock","owner":"sherlock","createTime":1623042469,"lastAccessTime":0,"retention":0,"sd":{"cols":[{"name":"id","type":"int","comment":null,"setName":true,"setType":true,"setComment":false},{"name":"name","type":"string","comment":null,"setName":true,"setType":true,"setComment":false},{"name":"gender","type":"string","comment":null,"setName":true,"setType":true,"setComment":false}],"location":"hdfs://localhost:9000/user/hive/warehouse/sherlock.db/t2","inputFormat":"org.apache.hadoop.mapred.TextInputFormat","outputFormat":"org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat","compressed":false,"numBuckets":-1,"serdeInfo":{"name":null,"serializationLib":"org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe","parameters":{"serialization.format":",","field.delim":","},"setName":false,"setParameters":true,"parametersSize":2,"setSerializationLib":true},"bucketCols":[],"sortCols":[],"parameters":{},"skewedInfo":{"skewedColNames":[],"skewedColValues":[],"skewedColValueLocationMaps":{},"setSkewedColNames":true,"setSkewedColValues":true,"setSkewedColValueLocationMaps":true,"skewedColNamesSize":0,"skewedColNamesIterator":[],"skewedColValuesSize":0,"skewedColValuesIterator":[],"skewedColValueLocationMapsSize":0},"storedAsSubDirectories":false,"colsSize":3,"colsIterator":[{"name":"id","type":"int","comment":null,"setName":true,"setType":true,"setComment":false},{"name":"name","type":"string","comment":null,"setName":true,"setType":true,"setComment":false},{"name":"gender","type":"string","comment":null,"setName":true,"setType":true,"setComment":false}],"setCompressed":true,"setNumBuckets":true,"bucketColsSize":0,"bucketColsIterator":[],"sortColsSize":0,"sortColsIterator":[],"setStoredAsSubDirectories":true,"setCols":true,"setLocation":true,"setInputFormat":true,"setOutputFormat":true,"setSerdeInfo":true,"setBucketCols":true,"setSortCols":true,"setSkewedInfo":true,"setParameters":true,"parametersSize":0},"partitionKeys":[],"parameters":{"totalSize":"0","numRows":"0","rawDataSize":"0","COLUMN_STATS_ACCURATE":"{\"BASIC_STATS\":\"true\"}","numFiles":"0","transient_lastDdlTime":"1623042469"},"viewOriginalText":null,"viewExpandedText":null,"tableType":"MANAGED_TABLE","privileges":null,"temporary":false,"rewriteEnabled":false,"partitionKeysSize":0,"setOwner":true,"setPartitionKeys":true,"setViewOriginalText":false,"setViewExpandedText":false,"setTableType":true,"setRetention":true,"partitionKeysIterator":[],"setTemporary":false,"setRewriteEnabled":true,"setDbName":true,"setCreateTime":true,"setTableName":true,"setSd":true,"setParameters":true,"setPrivileges":false,"setLastAccessTime":true,"parametersSize":6}
```

分析上面日志中的 input 和 output

input:

```json
{
    "tableName":"t1",
    "dbName":"sherlock",
    "owner":"sherlock",
    "createTime":1623042427,
    "lastAccessTime":0,
    "retention":0,
    "sd":{
        "cols":[
            {
                "name":"id",
                "type":"int",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            },
            {
                "name":"name",
                "type":"string",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            },
            {
                "name":"gender",
                "type":"string",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            }
        ],
        "location":"hdfs://localhost:9000/user/hive/warehouse/sherlock.db/t1",
        "inputFormat":"org.apache.hadoop.mapred.TextInputFormat",
        "outputFormat":"org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat",
        "compressed":false,
        "numBuckets":-1,
        "serdeInfo":{
            "name":null,
            "serializationLib":"org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe",
            "parameters":{
                "serialization.format":",",
                "field.delim":","
            },
            "setName":false,
            "setParameters":true,
            "parametersSize":2,
            "setSerializationLib":true
        },
        "bucketCols":[

        ],
        "sortCols":[

        ],
        "parameters":{

        },
        "skewedInfo":{
            "skewedColNames":[

            ],
            "skewedColValues":[

            ],
            "skewedColValueLocationMaps":{

            },
            "setSkewedColNames":true,
            "setSkewedColValues":true,
            "setSkewedColValueLocationMaps":true,
            "skewedColNamesSize":0,
            "skewedColNamesIterator":[

            ],
            "skewedColValuesSize":0,
            "skewedColValuesIterator":[

            ],
            "skewedColValueLocationMapsSize":0
        },
        "storedAsSubDirectories":false,
        "colsSize":3,
        "colsIterator":[
            {
                "name":"id",
                "type":"int",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            },
            {
                "name":"name",
                "type":"string",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            },
            {
                "name":"gender",
                "type":"string",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            }
        ],
        "setCompressed":true,
        "setNumBuckets":true,
        "bucketColsSize":0,
        "bucketColsIterator":[

        ],
        "sortColsSize":0,
        "sortColsIterator":[

        ],
        "setStoredAsSubDirectories":true,
        "setCols":true,
        "setLocation":true,
        "setInputFormat":true,
        "setOutputFormat":true,
        "setSerdeInfo":true,
        "setBucketCols":true,
        "setSortCols":true,
        "setSkewedInfo":true,
        "setParameters":true,
        "parametersSize":0
    },
    "partitionKeys":[

    ],
    "parameters":{
        "transient_lastDdlTime":"1623042628",
        "totalSize":"50",
        "numRows":"0",
        "rawDataSize":"0",
        "numFiles":"1"
    },
    "viewOriginalText":null,
    "viewExpandedText":null,
    "tableType":"MANAGED_TABLE",
    "privileges":null,
    "temporary":false,
    "rewriteEnabled":false,
    "partitionKeysSize":0,
    "setOwner":true,
    "setPartitionKeys":true,
    "setViewOriginalText":false,
    "setViewExpandedText":false,
    "setTableType":true,
    "setRetention":true,
    "partitionKeysIterator":[

    ],
    "setTemporary":false,
    "setRewriteEnabled":true,
    "setDbName":true,
    "setCreateTime":true,
    "setTableName":true,
    "setSd":true,
    "setParameters":true,
    "setPrivileges":false,
    "setLastAccessTime":true,
    "parametersSize":5
}
```

output:

```json
{
    "tableName":"t2",
    "dbName":"sherlock",
    "owner":"sherlock",
    "createTime":1623042469,
    "lastAccessTime":0,
    "retention":0,
    "sd":{
        "cols":[
            {
                "name":"id",
                "type":"int",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            },
            {
                "name":"name",
                "type":"string",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            },
            {
                "name":"gender",
                "type":"string",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            }
        ],
        "location":"hdfs://localhost:9000/user/hive/warehouse/sherlock.db/t2",
        "inputFormat":"org.apache.hadoop.mapred.TextInputFormat",
        "outputFormat":"org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat",
        "compressed":false,
        "numBuckets":-1,
        "serdeInfo":{
            "name":null,
            "serializationLib":"org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe",
            "parameters":{
                "serialization.format":",",
                "field.delim":","
            },
            "setName":false,
            "setParameters":true,
            "parametersSize":2,
            "setSerializationLib":true
        },
        "bucketCols":[

        ],
        "sortCols":[

        ],
        "parameters":{

        },
        "skewedInfo":{
            "skewedColNames":[

            ],
            "skewedColValues":[

            ],
            "skewedColValueLocationMaps":{

            },
            "setSkewedColNames":true,
            "setSkewedColValues":true,
            "setSkewedColValueLocationMaps":true,
            "skewedColNamesSize":0,
            "skewedColNamesIterator":[

            ],
            "skewedColValuesSize":0,
            "skewedColValuesIterator":[

            ],
            "skewedColValueLocationMapsSize":0
        },
        "storedAsSubDirectories":false,
        "colsSize":3,
        "colsIterator":[
            {
                "name":"id",
                "type":"int",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            },
            {
                "name":"name",
                "type":"string",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            },
            {
                "name":"gender",
                "type":"string",
                "comment":null,
                "setName":true,
                "setType":true,
                "setComment":false
            }
        ],
        "setCompressed":true,
        "setNumBuckets":true,
        "bucketColsSize":0,
        "bucketColsIterator":[

        ],
        "sortColsSize":0,
        "sortColsIterator":[

        ],
        "setStoredAsSubDirectories":true,
        "setCols":true,
        "setLocation":true,
        "setInputFormat":true,
        "setOutputFormat":true,
        "setSerdeInfo":true,
        "setBucketCols":true,
        "setSortCols":true,
        "setSkewedInfo":true,
        "setParameters":true,
        "parametersSize":0
    },
    "partitionKeys":[

    ],
    "parameters":{
        "totalSize":"0",
        "numRows":"0",
        "rawDataSize":"0",
        "COLUMN_STATS_ACCURATE":"{\"BASIC_STATS\":\"true\"}",
        "numFiles":"0",
        "transient_lastDdlTime":"1623042469"
    },
    "viewOriginalText":null,
    "viewExpandedText":null,
    "tableType":"MANAGED_TABLE",
    "privileges":null,
    "temporary":false,
    "rewriteEnabled":false,
    "partitionKeysSize":0,
    "setOwner":true,
    "setPartitionKeys":true,
    "setViewOriginalText":false,
    "setViewExpandedText":false,
    "setTableType":true,
    "setRetention":true,
    "partitionKeysIterator":[

    ],
    "setTemporary":false,
    "setRewriteEnabled":true,
    "setDbName":true,
    "setCreateTime":true,
    "setTableName":true,
    "setSd":true,
    "setParameters":true,
    "setPrivileges":false,
    "setLastAccessTime":true,
    "parametersSize":6
}
```



#### 3.1.2 DDL 操作

当我们在Hive的beeline客户端中创建一张表时，如下：

```sql
CREATE TABLE testposthook(
  id int COMMENT "id",
  name string COMMENT "姓名"
)COMMENT "建表_测试Hive Hooks"
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
LOCATION '/user/hive/warehouse/';
```

观察hive.log日志：

![img](https://segmentfault.com/img/remote/1460000023723863)

上面的 Hook metastore 输出值有两个：**第一个是数据库的元数据信息**，**第二个是表的元数据信息**

- 数据库元数据

```json
{
    "name":"default",
    "description":"Default Hive database",
    "locationUri":"hdfs://kms-1.apache.com:8020/user/hive/warehouse",
    "parameters":{

    },
    "privileges":null,
    "ownerName":"public",
    "ownerType":"ROLE",
    "setParameters":true,
    "parametersSize":0,
    "setOwnerName":true,
    "setOwnerType":true,
    "setPrivileges":false,
    "setName":true,
    "setDescription":true,
    "setLocationUri":true
}
```

- 表元数据

```json
{
    "tableName":"testposthook",
    "dbName":"default",
    "owner":"anonymous",
    "createTime":1597985444,
    "lastAccessTime":0,
    "retention":0,
    "sd":{
        "cols":[

        ],
        "location":null,
        "inputFormat":"org.apache.hadoop.mapred.SequenceFileInputFormat",
        "outputFormat":"org.apache.hadoop.hive.ql.io.HiveSequenceFileOutputFormat",
        "compressed":false,
        "numBuckets":-1,
        "serdeInfo":{
            "name":null,
            "serializationLib":"org.apache.hadoop.hive.serde2.MetadataTypedColumnsetSerDe",
            "parameters":{
                "serialization.format":"1"
            },
            "setSerializationLib":true,
            "setParameters":true,
            "parametersSize":1,
            "setName":false
        },
        "bucketCols":[

        ],
        "sortCols":[

        ],
        "parameters":{

        },
        "skewedInfo":{
            "skewedColNames":[

            ],
            "skewedColValues":[

            ],
            "skewedColValueLocationMaps":{

            },
            "skewedColNamesIterator":[

            ],
            "skewedColValuesSize":0,
            "skewedColValuesIterator":[

            ],
            "skewedColValueLocationMapsSize":0,
            "setSkewedColNames":true,
            "setSkewedColValues":true,
            "setSkewedColValueLocationMaps":true,
            "skewedColNamesSize":0
        },
        "storedAsSubDirectories":false,
        "colsSize":0,
        "setParameters":true,
        "parametersSize":0,
        "setOutputFormat":true,
        "setSerdeInfo":true,
        "setBucketCols":true,
        "setSortCols":true,
        "setSkewedInfo":true,
        "colsIterator":[

        ],
        "setCompressed":false,
        "setNumBuckets":true,
        "bucketColsSize":0,
        "bucketColsIterator":[

        ],
        "sortColsSize":0,
        "sortColsIterator":[

        ],
        "setStoredAsSubDirectories":false,
        "setCols":true,
        "setLocation":false,
        "setInputFormat":true
    },
    "partitionKeys":[

    ],
    "parameters":{

    },
    "viewOriginalText":null,
    "viewExpandedText":null,
    "tableType":"MANAGED_TABLE",
    "privileges":null,
    "temporary":false,
    "rewriteEnabled":false,
    "partitionKeysSize":0,
    "setDbName":true,
    "setSd":true,
    "setParameters":true,
    "setCreateTime":true,
    "setLastAccessTime":false,
    "parametersSize":0,
    "setTableName":true,
    "setPrivileges":false,
    "setOwner":true,
    "setPartitionKeys":true,
    "setViewOriginalText":false,
    "setViewExpandedText":false,
    "setTableType":true,
    "setRetention":false,
    "partitionKeysIterator":[

    ],
    "setTemporary":false,
    "setRewriteEnabled":false
}
```

我们发现上面的表元数据信息中，**cols[]**列没有数据，即没有建表时的字段`id`和字段`name`的信息。如果要获取这些信息，可以执行下面的命令：

```sh
ALTER TABLE testposthook;
ADD COLUMNS (age int COMMENT '年龄');
```

再次观察日志信息：

![img](https://segmentfault.com/img/remote/1460000023723861)

上面的日志中，Hook metastore 只有一个输入和一个输出：都表示 table 的元数据信息。

- 输入

```shell
{
    "tableName":"testposthook",
    "dbName":"default",
    "owner":"anonymous",
    "createTime":1597985445,
    "lastAccessTime":0,
    "retention":0,
    "sd":{
        "cols":[
            {
                "name":"id",
                "type":"int",
                "comment":"id",
                "setName":true,
                "setType":true,
                "setComment":true
            },
            {
                "name":"name",
                "type":"string",
                "comment":"姓名",
                "setName":true,
                "setType":true,
                "setComment":true
            }
        ],
        "location":"hdfs://kms-1.apache.com:8020/user/hive/warehouse",
        "inputFormat":"org.apache.hadoop.mapred.TextInputFormat",
        "outputFormat":"org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat",
        "compressed":false,
        "numBuckets":-1,
        "serdeInfo":{
            "name":null,
            "serializationLib":"org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe",
            "parameters":{
                "serialization.format":"    ",
                "field.delim":"    "
            },
            "setSerializationLib":true,
            "setParameters":true,
            "parametersSize":2,
            "setName":false
        },
        "bucketCols":[

        ],
        "sortCols":[

        ],
        "parameters":{

        },
        "skewedInfo":{
            "skewedColNames":[

            ],
            "skewedColValues":[

            ],
            "skewedColValueLocationMaps":{

            },
            "skewedColNamesIterator":[

            ],
            "skewedColValuesSize":0,
            "skewedColValuesIterator":[

            ],
            "skewedColValueLocationMapsSize":0,
            "setSkewedColNames":true,
            "setSkewedColValues":true,
            "setSkewedColValueLocationMaps":true,
            "skewedColNamesSize":0
        },
        "storedAsSubDirectories":false,
        "colsSize":2,
        "setParameters":true,
        "parametersSize":0,
        "setOutputFormat":true,
        "setSerdeInfo":true,
        "setBucketCols":true,
        "setSortCols":true,
        "setSkewedInfo":true,
        "colsIterator":[
            {
                "name":"id",
                "type":"int",
                "comment":"id",
                "setName":true,
                "setType":true,
                "setComment":true
            },
            {
                "name":"name",
                "type":"string",
                "comment":"姓名",
                "setName":true,
                "setType":true,
                "setComment":true
            }
        ],
        "setCompressed":true,
        "setNumBuckets":true,
        "bucketColsSize":0,
        "bucketColsIterator":[

        ],
        "sortColsSize":0,
        "sortColsIterator":[

        ],
        "setStoredAsSubDirectories":true,
        "setCols":true,
        "setLocation":true,
        "setInputFormat":true
    },
    "partitionKeys":[

    ],
    "parameters":{
        "transient_lastDdlTime":"1597985445",
        "comment":"建表_测试Hive Hooks",
        "totalSize":"0",
        "numFiles":"0"
    },
    "viewOriginalText":null,
    "viewExpandedText":null,
    "tableType":"MANAGED_TABLE",
    "privileges":null,
    "temporary":false,
    "rewriteEnabled":false,
    "partitionKeysSize":0,
    "setDbName":true,
    "setSd":true,
    "setParameters":true,
    "setCreateTime":true,
    "setLastAccessTime":true,
    "parametersSize":4,
    "setTableName":true,
    "setPrivileges":false,
    "setOwner":true,
    "setPartitionKeys":true,
    "setViewOriginalText":false,
    "setViewExpandedText":false,
    "setTableType":true,
    "setRetention":true,
    "partitionKeysIterator":[

    ],
    "setTemporary":false,
    "setRewriteEnabled":true
}
```

从上面的json中可以看出**"cols"**列的字段元数据信息，我们再来看一下输出json：

- 输出

```shell
{
    "tableName":"testposthook",
    "dbName":"default",
    "owner":"anonymous",
    "createTime":1597985445,
    "lastAccessTime":0,
    "retention":0,
    "sd":{
        "cols":[
            {
                "name":"id",
                "type":"int",
                "comment":"id",
                "setName":true,
                "setType":true,
                "setComment":true
            },
            {
                "name":"name",
                "type":"string",
                "comment":"姓名",
                "setName":true,
                "setType":true,
                "setComment":true
            }
        ],
        "location":"hdfs://kms-1.apache.com:8020/user/hive/warehouse",
        "inputFormat":"org.apache.hadoop.mapred.TextInputFormat",
        "outputFormat":"org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat",
        "compressed":false,
        "numBuckets":-1,
        "serdeInfo":{
            "name":null,
            "serializationLib":"org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe",
            "parameters":{
                "serialization.format":"    ",
                "field.delim":"    "
            },
            "setSerializationLib":true,
            "setParameters":true,
            "parametersSize":2,
            "setName":false
        },
        "bucketCols":[

        ],
        "sortCols":[

        ],
        "parameters":{

        },
        "skewedInfo":{
            "skewedColNames":[

            ],
            "skewedColValues":[

            ],
            "skewedColValueLocationMaps":{

            },
            "skewedColNamesIterator":[

            ],
            "skewedColValuesSize":0,
            "skewedColValuesIterator":[

            ],
            "skewedColValueLocationMapsSize":0,
            "setSkewedColNames":true,
            "setSkewedColValues":true,
            "setSkewedColValueLocationMaps":true,
            "skewedColNamesSize":0
        },
        "storedAsSubDirectories":false,
        "colsSize":2,
        "setParameters":true,
        "parametersSize":0,
        "setOutputFormat":true,
        "setSerdeInfo":true,
        "setBucketCols":true,
        "setSortCols":true,
        "setSkewedInfo":true,
        "colsIterator":[
            {
                "name":"id",
                "type":"int",
                "comment":"id",
                "setName":true,
                "setType":true,
                "setComment":true
            },
            {
                "name":"name",
                "type":"string",
                "comment":"姓名",
                "setName":true,
                "setType":true,
                "setComment":true
            }
        ],
        "setCompressed":true,
        "setNumBuckets":true,
        "bucketColsSize":0,
        "bucketColsIterator":[

        ],
        "sortColsSize":0,
        "sortColsIterator":[

        ],
        "setStoredAsSubDirectories":true,
        "setCols":true,
        "setLocation":true,
        "setInputFormat":true
    },
    "partitionKeys":[

    ],
    "parameters":{
        "transient_lastDdlTime":"1597985445",
        "comment":"建表_测试Hive Hooks",
        "totalSize":"0",
        "numFiles":"0"
    },
    "viewOriginalText":null,
    "viewExpandedText":null,
    "tableType":"MANAGED_TABLE",
    "privileges":null,
    "temporary":false,
    "rewriteEnabled":false,
    "partitionKeysSize":0,
    "setDbName":true,
    "setSd":true,
    "setParameters":true,
    "setCreateTime":true,
    "setLastAccessTime":true,
    "parametersSize":4,
    "setTableName":true,
    "setPrivileges":false,
    "setOwner":true,
    "setPartitionKeys":true,
    "setViewOriginalText":false,
    "setViewExpandedText":false,
    "setTableType":true,
    "setRetention":true,
    "partitionKeysIterator":[

    ],
    "setTemporary":false,
    "setRewriteEnabled":true
}
```

> 该`output`对象不包含新列`age`，它表示修改表之前的元数据信息

## 四、Metastore Listeners 基本使用

### 4.1 代码

具体实现代码如下：

```
public class CustomListener extends MetaStoreEventListener {
    private static final Logger LOGGER = LoggerFactory.getLogger(CustomListener.class);
    private static final ObjectMapper objMapper = new ObjectMapper();

    public CustomListener(Configuration config) {
        super(config);
        logWithHeader(" created ");
    }

    // 监听建表操作
    @Override
    public void onCreateTable(CreateTableEvent event) {
        logWithHeader(event.getTable());
    }
    // 监听修改表操作
    @Override
    public void onAlterTable(AlterTableEvent event) {
        logWithHeader(event.getOldTable());
        logWithHeader(event.getNewTable());
    }

    private void logWithHeader(Object obj) {
        LOGGER.info("[CustomListener][Thread: " + Thread.currentThread().getName() + "] | " + objToStr(obj));
    }

    private String objToStr(Object obj) {
        try {
            return objMapper.writeValueAsString(obj);
        } catch (IOException e) {
            LOGGER.error("Error on conversion", e);
        }
        return null;
    }
}
```

### 4.2 使用过程解释

使用方式与Hooks有一点不同，Hive Hook是与Hiveserver进行交互的，而Listener是与Metastore交互的，即Listener运行在Metastore进程中的。具体使用方式如下：

首先将jar包放在$HIVE_HOME/lib目录下，然后配置hive-site.xml文件，配置内容为：

```
<property>
    <name>hive.metastore.event.listeners</name>
    <value>com.jmx.hooks.CustomListener</value>
    <description/>
 </property>
```

配置完成之后，需要重新启动元数据服务：

```
bin/hive --service metastore &
```

#### 4.2.1 建表操作

```
CREATE TABLE testlistener(
  id int COMMENT "id",
  name string COMMENT "姓名"
)COMMENT "建表_测试Hive Listener"
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
LOCATION '/user/hive/warehouse/';
```

观察hive.log日志：

![img](https://segmentfault.com/img/remote/1460000023723862)

```
{
    "tableName":"testlistener",
    "dbName":"default",
    "owner":"anonymous",
    "createTime":1597989316,
    "lastAccessTime":0,
    "retention":0,
    "sd":{
        "cols":[
            {
                "name":"id",
                "type":"int",
                "comment":"id",
                "setComment":true,
                "setType":true,
                "setName":true
            },
            {
                "name":"name",
                "type":"string",
                "comment":"姓名",
                "setComment":true,
                "setType":true,
                "setName":true
            }
        ],
        "location":"hdfs://kms-1.apache.com:8020/user/hive/warehouse",
        "inputFormat":"org.apache.hadoop.mapred.TextInputFormat",
        "outputFormat":"org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat",
        "compressed":false,
        "numBuckets":-1,
        "serdeInfo":{
            "name":null,
            "serializationLib":"org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe",
            "parameters":{
                "serialization.format":"    ",
                "field.delim":"    "
            },
            "setSerializationLib":true,
            "setParameters":true,
            "parametersSize":2,
            "setName":false
        },
        "bucketCols":[

        ],
        "sortCols":[

        ],
        "parameters":{

        },
        "skewedInfo":{
            "skewedColNames":[

            ],
            "skewedColValues":[

            ],
            "skewedColValueLocationMaps":{

            },
            "setSkewedColNames":true,
            "setSkewedColValues":true,
            "setSkewedColValueLocationMaps":true,
            "skewedColNamesSize":0,
            "skewedColNamesIterator":[

            ],
            "skewedColValuesSize":0,
            "skewedColValuesIterator":[

            ],
            "skewedColValueLocationMapsSize":0
        },
        "storedAsSubDirectories":false,
        "setCols":true,
        "setOutputFormat":true,
        "setSerdeInfo":true,
        "setBucketCols":true,
        "setSortCols":true,
        "colsSize":2,
        "colsIterator":[
            {
                "name":"id",
                "type":"int",
                "comment":"id",
                "setComment":true,
                "setType":true,
                "setName":true
            },
            {
                "name":"name",
                "type":"string",
                "comment":"姓名",
                "setComment":true,
                "setType":true,
                "setName":true
            }
        ],
        "setCompressed":true,
        "setNumBuckets":true,
        "bucketColsSize":0,
        "bucketColsIterator":[

        ],
        "sortColsSize":0,
        "sortColsIterator":[

        ],
        "setStoredAsSubDirectories":true,
        "setParameters":true,
        "setLocation":true,
        "setInputFormat":true,
        "parametersSize":0,
        "setSkewedInfo":true
    },
    "partitionKeys":[

    ],
    "parameters":{
        "transient_lastDdlTime":"1597989316",
        "comment":"建表_测试Hive Listener",
        "totalSize":"0",
        "numFiles":"0"
    },
    "viewOriginalText":null,
    "viewExpandedText":null,
    "tableType":"MANAGED_TABLE",
    "privileges":{
        "userPrivileges":{
            "anonymous":[
                {
                    "privilege":"INSERT",
                    "createTime":-1,
                    "grantor":"anonymous",
                    "grantorType":"USER",
                    "grantOption":true,
                    "setGrantOption":true,
                    "setCreateTime":true,
                    "setGrantor":true,
                    "setGrantorType":true,
                    "setPrivilege":true
                },
                {
                    "privilege":"SELECT",
                    "createTime":-1,
                    "grantor":"anonymous",
                    "grantorType":"USER",
                    "grantOption":true,
                    "setGrantOption":true,
                    "setCreateTime":true,
                    "setGrantor":true,
                    "setGrantorType":true,
                    "setPrivilege":true
                },
                {
                    "privilege":"UPDATE",
                    "createTime":-1,
                    "grantor":"anonymous",
                    "grantorType":"USER",
                    "grantOption":true,
                    "setGrantOption":true,
                    "setCreateTime":true,
                    "setGrantor":true,
                    "setGrantorType":true,
                    "setPrivilege":true
                },
                {
                    "privilege":"DELETE",
                    "createTime":-1,
                    "grantor":"anonymous",
                    "grantorType":"USER",
                    "grantOption":true,
                    "setGrantOption":true,
                    "setCreateTime":true,
                    "setGrantor":true,
                    "setGrantorType":true,
                    "setPrivilege":true
                }
            ]
        },
        "groupPrivileges":null,
        "rolePrivileges":null,
        "setUserPrivileges":true,
        "setGroupPrivileges":false,
        "setRolePrivileges":false,
        "userPrivilegesSize":1,
        "groupPrivilegesSize":0,
        "rolePrivilegesSize":0
    },
    "temporary":false,
    "rewriteEnabled":false,
    "setParameters":true,
    "setPartitionKeys":true,
    "partitionKeysSize":0,
    "setSd":true,
    "setLastAccessTime":true,
    "setRetention":true,
    "partitionKeysIterator":[

    ],
    "parametersSize":4,
    "setTemporary":true,
    "setRewriteEnabled":false,
    "setTableName":true,
    "setDbName":true,
    "setOwner":true,
    "setViewOriginalText":false,
    "setViewExpandedText":false,
    "setTableType":true,
    "setPrivileges":true,
    "setCreateTime":true
}
```

当我们再执行修改表操作时

```
ALTER TABLE testlistener
 ADD COLUMNS (age int COMMENT '年龄');
```

再次观察日志：

![img](https://segmentfault.com/img/remote/1460000023723864)

可以看出上面有两条记录，第一条记录是old table的信息，第二条是修改之后的表的信息。

- old table

```
{
    "tableName":"testlistener",
    "dbName":"default",
    "owner":"anonymous",
    "createTime":1597989316,
    "lastAccessTime":0,
    "retention":0,
    "sd":{
        "cols":[
            {
                "name":"id",
                "type":"int",
                "comment":"id",
                "setComment":true,
                "setType":true,
                "setName":true
            },
            {
                "name":"name",
                "type":"string",
                "comment":"姓名",
                "setComment":true,
                "setType":true,
                "setName":true
            }
        ],
        "location":"hdfs://kms-1.apache.com:8020/user/hive/warehouse",
        "inputFormat":"org.apache.hadoop.mapred.TextInputFormat",
        "outputFormat":"org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat",
        "compressed":false,
        "numBuckets":-1,
        "serdeInfo":{
            "name":null,
            "serializationLib":"org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe",
            "parameters":{
                "serialization.format":"    ",
                "field.delim":"    "
            },
            "setSerializationLib":true,
            "setParameters":true,
            "parametersSize":2,
            "setName":false
        },
        "bucketCols":[

        ],
        "sortCols":[

        ],
        "parameters":{

        },
        "skewedInfo":{
            "skewedColNames":[

            ],
            "skewedColValues":[

            ],
            "skewedColValueLocationMaps":{

            },
            "setSkewedColNames":true,
            "setSkewedColValues":true,
            "setSkewedColValueLocationMaps":true,
            "skewedColNamesSize":0,
            "skewedColNamesIterator":[

            ],
            "skewedColValuesSize":0,
            "skewedColValuesIterator":[

            ],
            "skewedColValueLocationMapsSize":0
        },
        "storedAsSubDirectories":false,
        "setCols":true,
        "setOutputFormat":true,
        "setSerdeInfo":true,
        "setBucketCols":true,
        "setSortCols":true,
        "colsSize":2,
        "colsIterator":[
            {
                "name":"id",
                "type":"int",
                "comment":"id",
                "setComment":true,
                "setType":true,
                "setName":true
            },
            {
                "name":"name",
                "type":"string",
                "comment":"姓名",
                "setComment":true,
                "setType":true,
                "setName":true
            }
        ],
        "setCompressed":true,
        "setNumBuckets":true,
        "bucketColsSize":0,
        "bucketColsIterator":[

        ],
        "sortColsSize":0,
        "sortColsIterator":[

        ],
        "setStoredAsSubDirectories":true,
        "setParameters":true,
        "setLocation":true,
        "setInputFormat":true,
        "parametersSize":0,
        "setSkewedInfo":true
    },
    "partitionKeys":[

    ],
    "parameters":{
        "totalSize":"0",
        "numFiles":"0",
        "transient_lastDdlTime":"1597989316",
        "comment":"建表_测试Hive Listener"
    },
    "viewOriginalText":null,
    "viewExpandedText":null,
    "tableType":"MANAGED_TABLE",
    "privileges":null,
    "temporary":false,
    "rewriteEnabled":false,
    "setParameters":true,
    "setPartitionKeys":true,
    "partitionKeysSize":0,
    "setSd":true,
    "setLastAccessTime":true,
    "setRetention":true,
    "partitionKeysIterator":[

    ],
    "parametersSize":4,
    "setTemporary":false,
    "setRewriteEnabled":true,
    "setTableName":true,
    "setDbName":true,
    "setOwner":true,
    "setViewOriginalText":false,
    "setViewExpandedText":false,
    "setTableType":true,
    "setPrivileges":false,
    "setCreateTime":true
}
```

- new table

```
{
    "tableName":"testlistener",
    "dbName":"default",
    "owner":"anonymous",
    "createTime":1597989316,
    "lastAccessTime":0,
    "retention":0,
    "sd":{
        "cols":[
            {
                "name":"id",
                "type":"int",
                "comment":"id",
                "setComment":true,
                "setType":true,
                "setName":true
            },
            {
                "name":"name",
                "type":"string",
                "comment":"姓名",
                "setComment":true,
                "setType":true,
                "setName":true
            },
            {
                "name":"age",
                "type":"int",
                "comment":"年龄",
                "setComment":true,
                "setType":true,
                "setName":true
            }
        ],
        "location":"hdfs://kms-1.apache.com:8020/user/hive/warehouse",
        "inputFormat":"org.apache.hadoop.mapred.TextInputFormat",
        "outputFormat":"org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat",
        "compressed":false,
        "numBuckets":-1,
        "serdeInfo":{
            "name":null,
            "serializationLib":"org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe",
            "parameters":{
                "serialization.format":"    ",
                "field.delim":"    "
            },
            "setSerializationLib":true,
            "setParameters":true,
            "parametersSize":2,
            "setName":false
        },
        "bucketCols":[

        ],
        "sortCols":[

        ],
        "parameters":{

        },
        "skewedInfo":{
            "skewedColNames":[

            ],
            "skewedColValues":[

            ],
            "skewedColValueLocationMaps":{

            },
            "setSkewedColNames":true,
            "setSkewedColValues":true,
            "setSkewedColValueLocationMaps":true,
            "skewedColNamesSize":0,
            "skewedColNamesIterator":[

            ],
            "skewedColValuesSize":0,
            "skewedColValuesIterator":[

            ],
            "skewedColValueLocationMapsSize":0
        },
        "storedAsSubDirectories":false,
        "setCols":true,
        "setOutputFormat":true,
        "setSerdeInfo":true,
        "setBucketCols":true,
        "setSortCols":true,
        "colsSize":3,
        "colsIterator":[
            {
                "name":"id",
                "type":"int",
                "comment":"id",
                "setComment":true,
                "setType":true,
                "setName":true
            },
            {
                "name":"name",
                "type":"string",
                "comment":"姓名",
                "setComment":true,
                "setType":true,
                "setName":true
            },
            {
                "name":"age",
                "type":"int",
                "comment":"年龄",
                "setComment":true,
                "setType":true,
                "setName":true
            }
        ],
        "setCompressed":true,
        "setNumBuckets":true,
        "bucketColsSize":0,
        "bucketColsIterator":[

        ],
        "sortColsSize":0,
        "sortColsIterator":[

        ],
        "setStoredAsSubDirectories":true,
        "setParameters":true,
        "setLocation":true,
        "setInputFormat":true,
        "parametersSize":0,
        "setSkewedInfo":true
    },
    "partitionKeys":[

    ],
    "parameters":{
        "totalSize":"0",
        "last_modified_time":"1597989660",
        "numFiles":"0",
        "transient_lastDdlTime":"1597989660",
        "comment":"建表_测试Hive Listener",
        "last_modified_by":"anonymous"
    },
    "viewOriginalText":null,
    "viewExpandedText":null,
    "tableType":"MANAGED_TABLE",
    "privileges":null,
    "temporary":false,
    "rewriteEnabled":false,
    "setParameters":true,
    "setPartitionKeys":true,
    "partitionKeysSize":0,
    "setSd":true,
    "setLastAccessTime":true,
    "setRetention":true,
    "partitionKeysIterator":[

    ],
    "parametersSize":6,
    "setTemporary":false,
    "setRewriteEnabled":true,
    "setTableName":true,
    "setDbName":true,
    "setOwner":true,
    "setViewOriginalText":false,
    "setViewExpandedText":false,
    "setTableType":true,
    "setPrivileges":false,
    "setCreateTime":true
}
```

可以看出：修改之后的表的元数据信息中，包含新添加的列`age`。

## 五、总结

在本文中，我们介绍了如何在Hive中操作元数据，从而能够自动进行元数据管理。我们给出了Hive Hooks和Metastore  Listener的基本使用方式，这些方式可以帮助我们实现操作元数据。当然也可以将这些元数据信息推送到Kafka中，以此构建自己的元数据管理系统。