# 【Canal】Install

## 一、准备

- 对于自建 MySQL , 需要先开启 Binlog 写入功能，配置 binlog-format 为 ROW 模式，my.cnf 中配置如下

  ```sh
  vi /etc/my.cnf
  [mysqld]
  log-bin=mysql-bin # 开启 binlog
  binlog-format=ROW # 选择 ROW 模式
  server_id=1 # 配置 MySQL replaction 需要定义，不要和 canal 的 slaveId 重复
  ```

  - 注意：针对阿里云 RDS for MySQL , 默认打开了 binlog , 并且账号默认具有 binlog dump 权限 , 不需要任何权限或者 binlog 设置,可以直接跳过这一步

  重启 MySQL: `systemctl restart mysqld`

  然后登陆到 MySQL 之后，查看是否已经修改过来:

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

- 授权 canal 链接 MySQL 账号具有作为 MySQL slave 的权限, 如果已有账户可直接 grant

  ```sql
  CREATE USER 'canal'@'%' IDENTIFIED BY 'canal';  
  GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
  GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%';
  FLUSH PRIVILEGES;
  ```

## 二、安装 & 配置

- 访问 [release 页面](https://github.com/alibaba/canal/releases) , 选择需要的包下载, 如以 1.1.5 版本为例

  ```sh
  wget https://github.com/alibaba/canal/releases/download/canal-1.1.5/canal.deployer-1.1.5.tar.gz
  ```

  

- 解压缩

  ```sh
  mkdir /opt/canal
  tar -zxvf /home/hmd/caiyi/installation/canal/canal.deployer-1.1.5.tar.gz -C /opt/canal
  ```

  - 解压完成后，进入 /opt/canal 目录，可以看到如下结构

    ```sh
    drwxr-xr-x 2 jianghang jianghang  136 2013-02-05 21:51 bin
    drwxr-xr-x 4 jianghang jianghang  160 2013-02-05 21:51 conf
    drwxr-xr-x 2 jianghang jianghang 1.3K 2013-02-05 21:51 lib
    drwxr-xr-x 2 jianghang jianghang   48 2013-02-05 21:29 logs
    ```

- 配置修改

  Canal 的配置分为两个部分 instance.properties 和 canal.properties

  - instance.properties

    ```properties
    #################################################
    ## mysql serverId , v1.0.26+ will autoGen
    # canal.instance.mysql.slaveId=0
    canal.instance.mysql.slaveId=11269
    
    # enable gtid use true/false
    canal.instance.gtidon=false
    
    # position info
    canal.instance.master.address=127.0.0.1:3306
    canal.instance.master.journal.name=
    canal.instance.master.position=
    canal.instance.master.timestamp=
    canal.instance.master.gtid=
    
    # rds oss binlog
    canal.instance.rds.accesskey=
    canal.instance.rds.secretkey=
    canal.instance.rds.instanceId=
    
    # table meta tsdb info
    canal.instance.tsdb.enable=true
    #canal.instance.tsdb.url=jdbc:mysql://127.0.0.1:3306/canal_tsdb
    #canal.instance.tsdb.dbUsername=canal
    #canal.instance.tsdb.dbPassword=canal
    
    #canal.instance.standby.address =
    #canal.instance.standby.journal.name =
    #canal.instance.standby.position =
    #canal.instance.standby.timestamp =
    #canal.instance.standby.gtid=
    
    # username/password
    canal.instance.dbUsername=canal
    canal.instance.dbPassword=1234
    canal.instance.connectionCharset=UTF-8
    # enable druid Decrypt database password
    canal.instance.enableDruid=false
    #canal.instance.pwdPublicKey=MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBALK4BUxdDltRRE5/zXpVEVPUgunvscYFtEip3pmLlhrWpacX7y7GCMo2/JM6LeHmiiNdH1FWgGCpUfircSwlWKUCAwEAAQ==
    
    # table regex
    canal.instance.filter.regex=.*\\..*
    # table black regex
    canal.instance.filter.black.regex=mysql\\.slave_.*
    # table field filter(format: schema1.tableName1:field1/field2,schema2.tableName2:field1/field2)
    #canal.instance.filter.field=test1.t_product:id/subject/keywords,test2.t_company:id/name/contact/ch
    # table field black filter(format: schema1.tableName1:field1/field2,schema2.tableName2:field1/field2)
    #canal.instance.filter.black.field=test1.t_product:subject/product_image,test2.t_company:id/name/contact/ch
    
    # mq config
    canal.mq.topic=example
    # dynamic topic route by schema or table regex
    canal.mq.dynamicTopic=iris_hqls:iris\\.hqls,test_canal:sherlock\\.test
    canal.mq.partition=0
    # hash partition config
    #canal.mq.partitionsNum=3
    #canal.mq.partitionHash=test.table:id^name,.*\\..*
    #canal.mq.dynamicTopicPartitionNum=test.*:4,mycanal:6
    #################################################
    ```

    canal.instance.connectionCharset 代表数据库的编码方式对应到 java 中的编码类型，比如 UTF-8，GBK , ISO-8859-1

    如果系统是1个 cpu，需要将 canal.instance.parser.parallel 设置为 false

  - canal.properties

    ```properties
    #################################################
    ######### 		common argument		#############
    #################################################
    # tcp bind ip
    canal.ip =
    # register ip to zookeeper
    canal.register.ip =
    canal.port = 11111
    canal.metrics.pull.port = 11112
    # canal instance user/passwd
    # canal.user = canal
    # canal.passwd = E3619321C1A937C46A0D8BD1DAC39F93B27D4458
    
    # canal admin config
    #canal.admin.manager = 127.0.0.1:8089
    canal.admin.port = 11110
    canal.admin.user = admin
    canal.admin.passwd = 4ACFE3202A5FF5CF467898FC58AAB1D615029441
    # admin auto register
    #canal.admin.register.auto = true
    #canal.admin.register.cluster =
    #canal.admin.register.name =
    
    canal.zkServers =
    # flush data to zk
    canal.zookeeper.flush.period = 1000
    canal.withoutNetty = false
    # tcp, kafka, rocketMQ, rabbitMQ
    canal.serverMode = kafka
    # flush meta cursor/parse position to file
    canal.file.data.dir = ${canal.conf.dir}
    canal.file.flush.period = 1000
    ## memory store RingBuffer size, should be Math.pow(2,n)
    canal.instance.memory.buffer.size = 16384
    ## memory store RingBuffer used memory unit size , default 1kb
    canal.instance.memory.buffer.memunit = 1024 
    ## meory store gets mode used MEMSIZE or ITEMSIZE
    canal.instance.memory.batch.mode = MEMSIZE
    canal.instance.memory.rawEntry = true
    
    ## detecing config
    canal.instance.detecting.enable = false
    #canal.instance.detecting.sql = insert into retl.xdual values(1,now()) on duplicate key update x=now()
    canal.instance.detecting.sql = select 1
    canal.instance.detecting.interval.time = 3
    canal.instance.detecting.retry.threshold = 3
    canal.instance.detecting.heartbeatHaEnable = false
    
    # support maximum transaction size, more than the size of the transaction will be cut into multiple transactions delivery
    canal.instance.transaction.size =  1024
    # mysql fallback connected to new master should fallback times
    canal.instance.fallbackIntervalInSeconds = 60
    
    # network config
    canal.instance.network.receiveBufferSize = 16384
    canal.instance.network.sendBufferSize = 16384
    canal.instance.network.soTimeout = 30
    
    # binlog filter config
    canal.instance.filter.druid.ddl = true
    canal.instance.filter.query.dcl = true
    canal.instance.filter.query.dml = false
    canal.instance.filter.query.ddl = false
    canal.instance.filter.table.error = false
    canal.instance.filter.rows = false
    canal.instance.filter.transaction.entry = false
    canal.instance.filter.dml.insert = false
    canal.instance.filter.dml.update = false
    canal.instance.filter.dml.delete = false
    
    # binlog format/image check
    canal.instance.binlog.format = ROW,STATEMENT,MIXED 
    canal.instance.binlog.image = FULL,MINIMAL,NOBLOB
    
    # binlog ddl isolation
    canal.instance.get.ddl.isolation = false
    
    # parallel parser config
    canal.instance.parser.parallel = true
    ## concurrent thread number, default 60% available processors, suggest not to exceed Runtime.getRuntime().availableProcessors()
    #canal.instance.parser.parallelThreadSize = 16
    ## disruptor ringbuffer size, must be power of 2
    canal.instance.parser.parallelBufferSize = 256
    
    # table meta tsdb info
    canal.instance.tsdb.enable = true
    canal.instance.tsdb.dir = ${canal.file.data.dir:../conf}/${canal.instance.destination:}
    canal.instance.tsdb.url = jdbc:h2:${canal.instance.tsdb.dir}/h2;CACHE_SIZE=1000;MODE=MYSQL;
    canal.instance.tsdb.dbUsername = canal
    canal.instance.tsdb.dbPassword = canal
    # dump snapshot interval, default 24 hour
    canal.instance.tsdb.snapshot.interval = 24
    # purge snapshot expire , default 360 hour(15 days)
    canal.instance.tsdb.snapshot.expire = 360
    
    #################################################
    ######### 		destinations		#############
    #################################################
    canal.destinations = example
    # conf root dir
    canal.conf.dir = ../conf
    # auto scan instance dir add/remove and start/stop instance
    canal.auto.scan = true
    canal.auto.scan.interval = 5
    # set this value to 'true' means that when binlog pos not found, skip to latest.
    # WARN: pls keep 'false' in production env, or if you know what you want.
    canal.auto.reset.latest.pos.mode = false
    
    canal.instance.tsdb.spring.xml = classpath:spring/tsdb/h2-tsdb.xml
    #canal.instance.tsdb.spring.xml = classpath:spring/tsdb/mysql-tsdb.xml
    
    canal.instance.global.mode = spring
    canal.instance.global.lazy = false
    canal.instance.global.manager.address = ${canal.admin.manager}
    #canal.instance.global.spring.xml = classpath:spring/memory-instance.xml
    canal.instance.global.spring.xml = classpath:spring/file-instance.xml
    #canal.instance.global.spring.xml = classpath:spring/default-instance.xml
    
    ##################################################
    ######### 	      MQ Properties      #############
    ##################################################
    # aliyun ak/sk , support rds/mq
    canal.aliyun.accessKey =
    canal.aliyun.secretKey =
    canal.aliyun.uid=
    
    canal.mq.flatMessage = true
    canal.mq.canalBatchSize = 50
    canal.mq.canalGetTimeout = 100
    # Set this value to "cloud", if you want open message trace feature in aliyun.
    canal.mq.accessChannel = local
    
    canal.mq.database.hash = true
    canal.mq.send.thread.size = 30
    canal.mq.build.thread.size = 8
    
    ##################################################
    ######### 		     Kafka 		     #############
    ##################################################
    kafka.bootstrap.servers = 10.10.110.235:9092,10.10.110.236:9092,10.10.110.238:9092
    kafka.acks = all
    kafka.compression.type = none
    kafka.batch.size = 16384
    kafka.linger.ms = 1
    kafka.max.request.size = 1048576
    kafka.buffer.memory = 33554432
    kafka.max.in.flight.requests.per.connection = 1
    kafka.retries = 3
    
    kafka.kerberos.enable = false
    kafka.kerberos.krb5.file = "../conf/kerberos/krb5.conf"
    kafka.kerberos.jaas.file = "../conf/kerberos/jaas.conf"
    
    ##################################################
    ######### 		    RocketMQ	     #############
    ##################################################
    rocketmq.producer.group = test
    rocketmq.enable.message.trace = false
    rocketmq.customized.trace.topic =
    rocketmq.namespace =
    rocketmq.namesrv.addr = 127.0.0.1:9876
    rocketmq.retry.times.when.send.failed = 0
    rocketmq.vip.channel.enabled = false
    rocketmq.tag = 
    
    ##################################################
    ######### 		    RabbitMQ	     #############
    ##################################################
    rabbitmq.host =
    rabbitmq.virtual.host =
    rabbitmq.exchange =
    rabbitmq.username =
    rabbitmq.password =
    rabbitmq.deliveryMode =
    ```

    

## 三、启动 & 停止

- 启动

  ```
  sh bin/startup.sh
  ```

- 查看 server 日志

  ```
  tail -f logs/canal/canal.log</pre>
  ```

  ```
  2013-02-05 22:45:27.967 [main] INFO  com.alibaba.otter.canal.deployer.CanalLauncher - ## start the canal server.
  2013-02-05 22:45:28.113 [main] INFO  com.alibaba.otter.canal.deployer.CanalController - ## start the canal server[10.1.29.120:11111]
  2013-02-05 22:45:28.210 [main] INFO  com.alibaba.otter.canal.deployer.CanalLauncher - ## the canal server is running now ......
  ```

- 查看 instance 的日志

  ```
  vi logs/example/example.log
  ```

  ```
  2013-02-05 22:50:45.636 [main] INFO  c.a.o.c.i.spring.support.PropertyPlaceholderConfigurer - Loading properties file from class path resource [canal.properties]
  2013-02-05 22:50:45.641 [main] INFO  c.a.o.c.i.spring.support.PropertyPlaceholderConfigurer - Loading properties file from class path resource [example/instance.properties]
  2013-02-05 22:50:45.803 [main] INFO  c.a.otter.canal.instance.spring.CanalInstanceWithSpring - start CannalInstance for 1-example 
  2013-02-05 22:50:45.810 [main] INFO  c.a.otter.canal.instance.spring.CanalInstanceWithSpring - start successful....
  ```

- 关闭

  ```
  sh bin/stop.sh
  ```

## 