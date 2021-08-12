# 【Maxwell】安装

Maxwell 是一个能实时读取 MySQL 二进制日志 binlog，并生成 JSON 格式的消息，作为生产者发送给 Kafka，Kinesis、RabbitMQ、Redis、Google Cloud Pub/Sub、文件或其它平台的应用程序。它的常见应用场景有ETL、维护缓存、收集表级别的dml指标、增量到搜索引擎、数据分区迁移、切库binlog回滚方案等。官网(http://maxwells-daemon.io)、GitHub(https://github.com/zendesk/maxwell)

Maxwell 主要提供了下列功能：

- 支持 SELECT * FROM table 的方式进行全量数据初始化
- 支持在主库发生 failover 后，自动恢复 binlog 位置(GTID)
- 可以对数据进行分区，解决数据倾斜问题，发送到 kafka 的数据支持 database、table、column 等级别的数据分区
- 工作方式是伪装为 Slave，接收 binlog events，然后根据 schemas 信息拼装，可以接受 ddl、xid、row 等各种 event
- 除了 Maxwell 外，目前常用的 MySQL Binlog 解析工具主要有阿里的canal、mysql_streamer，三个工具对比如下：

<table>
  <thead>
    <tr>
      <th></th>
      <th>Canal（服务端）</th>
      <th>Maxwell（服务端和客户端）</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>语言</td>
      <td>Java</td>
      <td>Java</td>
    </tr>
    <tr>
      <td>活跃度</td>
      <td>活跃</td>
      <td>活跃</td>
    </tr>
    <tr>
      <td>HA</td>
      <td>支持</td>
      <td>定制但是支持断点还原</td>
    </tr>
    <tr>
      <td>数据落地</td>
      <td>定制</td>
      <td>落地到kafka</td>
    </tr>
    <tr>
      <td>分区</td>
      <td>支持</td>
      <td>支持</td>
    </tr>
    <tr>
      <td>bootstrap</td>
      <td>不支持</td>
      <td>支持</td>
    </tr>
    <tr>
      <td>数据格式</td>
      <td>格式自由</td>
      <td>json(格式固定)</td>
    </tr>
    <tr>
      <td>文档</td>
      <td>较详细</td>
      <td>较详细</td>
    </tr>
    <tr>
      <td>随机读</td>
      <td>支持</td>
      <td>支持</td>
    </tr>
  </tbody>
</table>

## 一、前置工作

安装 Maxwell 需要 MySQL 和 Kafka

1. 安装MySQL, 并修改my.cnf文件

   ```cnf
   # 添加如下内容
   vi /etc/my.cnf
   [mysqld]
   server-id=1
   binlog_format=ROW
   log-bin=mysql-bin
   ```

   重启 MySQL,然后登陆到 MySQL 之后，查看是否已经修改过来:

   ```sql
   mysql> show variables like 'binlog_format';
   +---------------+-------+
   | Variable_name | Value |
   +---------------+-------+
   | binlog_format | ROW   |
   +---------------+-------+
   
   mysql> show variables like '%log_bin%';
   +---------------------------------+---------------------------------------+
   | Variable_name                   | Value                                 |
   +---------------------------------+---------------------------------------+
   | log_bin                         | ON                                    |
   | log_bin_basename                | /usr/local/mysql/data/mysql-bin       |
   | log_bin_index                   | /usr/local/mysql/data/mysql-bin.index |
   | log_bin_trust_function_creators | OFF                                   |
   | log_bin_use_v1_row_events       | OFF                                   |
   | sql_log_bin                     | ON                                    |
   +---------------------------------+---------------------------------------+
   ```

   在 MySQL 中添加 Maxwell 用户, 以及分配权限:

   ```sql
   mysql> create database maxwell;
   mysql> CREATE USER 'maxwell'@'%' IDENTIFIED BY '1234';
   mysql> GRANT ALL ON maxwell.* TO 'maxwell'@'%';
   mysql> GRANT SELECT, REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'maxwell'@'%';
   mysql> flush privileges;
   ```

2. 安装 Kafka, Zookeeper

   参考 Kafka, Zookeeper 安装流程, 正常安装测试即可

## 二、Maxwell 安装

```sh
tar -zxvf ~/Downloads/maxwell-1.33.1.tar.gz -C ~/devTools/opt

# sherlock @ mbp in ~ [19:39:21]
$ cd ~/devTools/opt/maxwell-1.33.1

# sherlock @ mbp in ~/devTools/opt/maxwell-1.33.1 [20:53:15]
$ pwd
/Users/sherlock/devTools/opt/maxwell-1.33.1
```

修改环境

```shell
# sherlock @ mbp in /usr/local [20:55:49] C:1
$ sudo ln -s /Users/sherlock/devTools/opt/maxwell-1.33.1 maxwell
```

添加配置文件 config.properties

```properties
# tl;dr config

# ********** binlog **********
log_level=INFO
producer=kafka
host=localhost
port=3306
user=maxwell
password=1234

# *********** kafka **********
kafka_topic=test_maxwell
kafka_partition_hash=murmur3
kafka_key_format=hash
producer_partition_by=table
kafka.compression.type=snappy
kafka.retries=5
kafka.acks=alls

# ********** output format stuff **********
output_binlog_position=true
output_nulls=true
output_server_id=true
output_thread_id=true
output_schema_id=true
# 需要MySQL开启 set global binlog_rows_query_log_events=on; 
output_row_query=true
output_primary_keys=true
output_primary_key_columns=true
output_commit_info=true
output_ddl=false

filter=exclude: *.*, include: sherlock.test

bootstrapper=async

#     *** general ***
# choose where to produce data to. stdout|file|kafka|kinesis|pubsub|sqs|rabbitmq|redis
#producer=kafka

# set the log level.  note that you can configure things further in log4j2.xml
#log_level=DEBUG # [DEBUG, INFO, WARN, ERROR]

# if set, maxwell will look up the scoped environment variables, strip off the prefix and inject the configs
#env_config_prefix=MAXWELL_

#     *** mysql ***

# mysql host to connect to
#host=hostname

# mysql port to connect to
#port=3306

# mysql user to connect as.  This user must have REPLICATION SLAVE permissions,
# as well as full access to the `maxwell` (or schema_database) database
#user=maxwell

# mysql password
#password=maxwell

# options to pass into the jdbc connection, given as opt=val&opt2=val2
#jdbc_options=opt1=100&opt2=hello

# name of the mysql database where maxwell keeps its own state
#schema_database=maxwell

# whether to use GTID or not for positioning
#gtid_mode=true

# SSL/TLS options
# To use VERIFY_CA or VERIFY_IDENTITY, you must set the trust store with Java opts:
#   -Djavax.net.ssl.trustStore=<truststore> -Djavax.net.ssl.trustStorePassword=<password>
# or import the MySQL cert into the global Java cacerts.
# MODE must be one of DISABLED, PREFERRED, REQUIRED, VERIFY_CA, or VERIFY_IDENTITY
#
# turns on ssl for the maxwell-store connection, other connections inherit this setting unless specified
#ssl=DISABLED
# for binlog-connector
#replication_ssl=DISABLED
# for the schema-capture connection, if used
#schema_ssl=DISABLED

# maxwell can optionally replicate from a different server than where it stores
# schema and binlog position info.  Specify that different server here:

#replication_host=other
#replication_user=username
#replication_password=password
#replication_port=3306

# This may be useful when using MaxScale's binlog mirroring host.
# Specifies that Maxwell should capture schema from a different server than
# it replicates from:

#schema_host=other
#schema_user=username
#schema_password=password
#schema_port=3306

#       *** output format ***

# records include binlog position (default false)
#output_binlog_position=true

# records include a gtid string (default false)
#output_gtid_position=true

# records include fields with null values (default true).  If this is false,
# fields where the value is null will be omitted entirely from output.
#output_nulls=true

# records include server_id (default false)
#output_server_id=true

# records include thread_id (default false)
#output_thread_id=true

# records include schema_id (default false)
#output_schema_id=true

# records include row query, binlog option "binlog_rows_query_log_events" must be enabled" (default false)
#output_row_query=true

# DML records include list of values that make up a row's primary key (default false)
#output_primary_keys=true

# DML records include list of columns that make up a row's primary key (default false)
#output_primary_key_columns=true

# records include commit and xid (default true)
#output_commit_info=true

# This controls whether maxwell will output JSON information containing
# DDL (ALTER/CREATE TABLE/ETC) infromation. (default: false)
# See also: ddl_kafka_topic
#output_ddl=true

#       *** kafka ***

# list of kafka brokers
#kafka.bootstrap.servers=hosta:9092,hostb:9092

# kafka topic to write to
# this can be static, e.g. 'maxwell', or dynamic, e.g. namespace_%{database}_%{table}
# in the latter case 'database' and 'table' will be replaced with the values for the row being processed
#kafka_topic=maxwell

# alternative kafka topic to write DDL (alter/create/drop) to.  Defaults to kafka_topic
#ddl_kafka_topic=maxwell_ddl

# hash function to use.  "default" is just the JVM's 'hashCode' function.
#kafka_partition_hash=default # [default, murmur3]

# how maxwell writes its kafka key.
#
# 'hash' looks like:
# {"database":"test","table":"tickets","pk.id":10001}
#
# 'array' looks like:
# ["test","tickets",[{"id":10001}]]
#
# default: "hash"
#kafka_key_format=hash # [hash, array]

# extra kafka options.  Anything prefixed "kafka." will get
# passed directly into the kafka-producer's config.

# a few defaults.
# These are 0.11-specific. They may or may not work with other versions.
#kafka.batch.size=16384


# kafka+SSL example
# kafka.security.protocol=SSL
# kafka.ssl.truststore.location=/var/private/ssl/kafka.client.truststore.jks
# kafka.ssl.truststore.password=test1234
# kafka.ssl.keystore.location=/var/private/ssl/kafka.client.keystore.jks
# kafka.ssl.keystore.password=test1234
# kafka.ssl.key.password=test1234#

# controls a heuristic check that maxwell may use to detect messages that
# we never heard back from.  The heuristic check looks for "stuck" messages, and
# will timeout maxwell after this many milliseconds.
#
# See https://github.com/zendesk/maxwell/blob/master/src/main/java/com/zendesk/maxwell/producer/InflightMessageList.java
# if you really want to get into it.
#producer_ack_timeout=120000 # default 0


#           *** partitioning ***

# What part of the data do we partition by?
#producer_partition_by=database # [database, table, primary_key, transaction_id, column]

# specify what fields to partition by when using producer_partition_by=column
# column separated list.
#producer_partition_columns=id,foo,bar

# when using producer_partition_by=column, partition by this when
# the specified column(s) don't exist.
#producer_partition_by_fallback=database

#            *** kinesis ***

#kinesis_stream=maxwell

# AWS places a 256 unicode character limit on the max key length of a record
# http://docs.aws.amazon.com/kinesis/latest/APIReference/API_PutRecord.html
#
# Setting this option to true enables hashing the key with the md5 algorithm
# before we send it to kinesis so all the keys work within the key size limit.
# Values: true, false
# Default: false
#kinesis_md5_keys=true

#            *** sqs ***

#sqs_queue_uri=aws_sqs_queue_uri

# The sqs producer will need aws credentials configured in the default
# root folder and file format. Please check below link on how to do it.
# http://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/setup-credentials.html

#            *** pub/sub ***

#pubsub_project_id=maxwell
#pubsub_topic=maxwell
#ddl_pubsub_topic=maxwell_ddl

#            *** rabbit-mq ***

#rabbitmq_host=rabbitmq_hostname
#rabbitmq_port=5672
#rabbitmq_user=guest
#rabbitmq_pass=guest
#rabbitmq_virtual_host=/
#rabbitmq_exchange=maxwell
#rabbitmq_exchange_type=fanout
#rabbitmq_exchange_durable=false
#rabbitmq_exchange_autodelete=false
#rabbitmq_routing_key_template=%db%.%table%
#rabbitmq_message_persistent=false
#rabbitmq_declare_exchange=true

#           *** redis ***

#redis_host=redis_host
#redis_port=6379
#redis_auth=redis_auth
#redis_database=0

# name of pubsub/list/whatever key to publish to
#redis_key=maxwell

# this can be static, e.g. 'maxwell', or dynamic, e.g. namespace_%{database}_%{table}
#redis_pub_channel=maxwell
# this can be static, e.g. 'maxwell', or dynamic, e.g. namespace_%{database}_%{table}
#redis_list_key=maxwell
# this can be static, e.g. 'maxwell', or dynamic, e.g. namespace_%{database}_%{table}
# Valid values for redis_type = pubsub|lpush. Defaults to pubsub

#redis_type=pubsub

#           *** custom producer ***

# the fully qualified class name for custom ProducerFactory
# see the following link for more details.
# http://maxwells-daemon.io/producers/#custom-producer
#custom_producer.factory=

# custom producer properties can be configured using the custom_producer.* property namespace
#custom_producer.custom_prop=foo

#          *** filtering ***

# filter rows out of Maxwell's output.  Command separated list of filter-rules, evaluated in sequence.
# A filter rule is:
#  <type> ":" <db> "." <tbl> [ "." <col> "=" <col_val> ]
#  type    ::= [ "include" | "exclude" | "blacklist" ]
#  db      ::= [ "/regexp/" | "string" | "`string`" | "*" ]
#  tbl     ::= [ "/regexp/" | "string" | "`string`" | "*" ]
#  col_val ::= "column_name"
#  tbl     ::= [ "/regexp/" | "string" | "`string`" | "*" ]
#
# See http://maxwells-daemon.io/filtering for more details
#
#filter= exclude: *.*, include: foo.*, include: bar.baz, include: foo.bar.col_eg = "value_to_match"

# javascript filter
# maxwell can run a bit of javascript for each row if you need very custom filtering/data munging.
# See http://maxwells-daemon.io/filtering/#javascript_filters for more details
#
#javascript=/path/to/javascript_filter_file

#       *** encryption ***

# Encryption mode. Possible values are none, data, and all. (default none)
#encrypt=none

# Specify the secret key to be used
#secret_key=RandomInitVector

#       *** monitoring ***

# Maxwell collects metrics via dropwizard. These can be exposed through the
# base logging mechanism (slf4j), JMX, HTTP or pushed to Datadog.
# Options: [jmx, slf4j, http, datadog]
# Supplying multiple is allowed.
#metrics_type=jmx,slf4j

# The prefix maxwell will apply to all metrics
#metrics_prefix=MaxwellMetrics # default MaxwellMetrics

# Enable (dropwizard) JVM metrics, default false
#metrics_jvm=true

# When metrics_type includes slf4j this is the frequency metrics are emitted to the log, in seconds
#metrics_slf4j_interval=60

# When metrics_type includes http or diagnostic is enabled, this is the port the server will bind to.
#http_port=8080

# When metrics_type includes http or diagnostic is enabled, this is the http path prefix, default /.
#http_path_prefix=/some/path/

# ** The following are Datadog specific. **
# When metrics_type includes datadog this is the way metrics will be reported.
# Options: [udp, http]
# Supplying multiple is not allowed.
#metrics_datadog_type=udp

# datadog tags that should be supplied
#metrics_datadog_tags=tag1:value1,tag2:value2

# The frequency metrics are pushed to datadog, in seconds
#metrics_datadog_interval=60

# required if metrics_datadog_type = http
#metrics_datadog_apikey=API_KEY

# required if metrics_datadog_type = udp
#metrics_datadog_host=localhost # default localhost
#metrics_datadog_port=8125 # default 8125

# Maxwell exposes http diagnostic endpoint to check below in parallel:
# 1. binlog replication lag
# 2. producer (currently kafka) lag

# To enable Maxwell diagnostic
#http_diagnostic=true # default false

# Diagnostic check timeout in milliseconds, required if diagnostic = true
#http_diagnostic_timeout=10000 # default 10000

#    *** misc ***

# maxwell's bootstrapping functionality has a couple of modes.
#
# In "async" mode, maxwell will output the replication stream while it
# simultaneously outputs the database to the topic.  Note that it won't
# output replication data for any tables it is currently bootstrapping -- this
# data will be buffered and output after the bootstrap is complete.
#
# In "sync" mode, maxwell stops the replication stream while it
# outputs bootstrap data.
#
# async mode keeps ops live while bootstrapping, but carries the possibility of
# data loss (due to buffering transactions).  sync mode is safer but you
# have to stop replication.
#bootstrapper=async [sync, async, none]

# output filename when using the "file" producer
#output_file=/path/to/file
```



