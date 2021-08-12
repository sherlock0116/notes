## 【Hive】本地单节点安装



前提安装配置好MySQL, 并测试 MySQL服务正常.

### 准备

1. 解压 tar 包, 建立软连接

   ```sh
   # 1. tar 解压到 ~/devTools/opt 目录下, 重命名
   ➜  ~ tar -zxvf apache-hive-2.3.8-bin.tar.gz -C ~/devTools/opt
   ➜  ~ mv ~/devTools/opt/apache-hive-2.3.8-bin ~/devTools/opt/hive-2.3.8
   
   # 2. 建立软连接
   ➜  ~ sudo ln -s ~/devTools/opt/hive-2.3.8 /usr/local/hive
   
   # 3. 修改环境变量(修改 ~/.bash_profile,添加如下内容)
   export HIVE_HOME=/usr/local/hive
   export PATH=$HIVE_HOME:$PATH
   ```

   

2. HDFS创建Hive仓库和tmp目录

   ```sh
   ➜  ~ hdfs dfs -mkdir -p /user/hive/warehouse
   ➜  ~ hdfs dfs -mkdir /tmp
   ```

   > `Tips`
   >
   > ​	HDFS 上 Hive 仓库的路径是 ***/hive/warehouse*** 




### 配置文件

这里要注意: 在复制 hive-default.xml.template 的时候,要将其复制改名为 hive-site.xml, 如果改名为 hive-default.xml ,则你在里面所修改的任何配置均不生效

#### hive-env.sh

```sh
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home
export HADOOP_HOME=/Users/sherlock/devTools/opt/hadoop-2.7.7
export HIVE_CONF_DIR=/Users/sherlock/devTools/opt/hive-2.3.7/conf
# 这个配置时暂未配置 HBase
# export HIVE_AUX_JARS_PATH=
```

#### hive-site.xml

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<!--
   Licensed to the Apache Software Foundation (ASF) under one or more
   contributor license agreements.  See the NOTICE file distributed with
   this work for additional information regarding copyright ownership.
   The ASF licenses this file to You under the Apache License, Version 2.0
   (the "License"); you may not use this file except in compliance with
   the License.  You may obtain a copy of the License at
       http://www.apache.org/licenses/LICENSE-2.0
   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
-->

<configuration>

	<property>
		<name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://localhost:3306/metastore_hive?createDatabaseIfNotExist=true&characterEncoding=UTF-8&useSSL=false</value>
  </property>

  <property>
  	<name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.jdbc.Driver</value>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>root</value>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>1234</value>
  </property>

  <property>
    <name>hive.server2.thrift.port</name>
    <value>10001</value>
  </property>

  <!-- 打印表时,显示字段名 -->
  <property>
    <name>hive.cli.print.header</name>
    <value>true</value>
  </property>

  <!-- 在交互窗口中,显示当前所在库 -->
 	<property>
    <name>hive.cli.print.current.db</name>
    <value>true</value>
  </property>

  <property>
    <name>hbase.zookeeper.quorum</name>
    <value>localhost</value>
  </property>
  	
  <property>
    <name>hive.metastore.schema.verification</name>
    <value>false</value>
  </property>

  <!-- 首次启动时需要创建 schema, 设置为 true允许自动创建, 否则会创建失败 -->
  <property>
    <name>datanucleus.schema.autoCreateAll</name>
    <value>true</value>
  </property>

		<!-- Hiveserver2已经不再需要hive.metastore.local这个配置项了
         hive.metastore.uris为空，则表示是metastore在本地，否则就是远程
         远程的话直接配置hive.metastore.uris即可 -->
    <!--<property>
        <name>hive.metastore.uris</name>
        <value>thrift://localhost:9083</value>
        <description>Thrift URI for the remote metastore. Used by metastore client to connect to remote metastore.</description>
    </property>-->
   

</configuration>
```



#### hive-log4j2.properties

```properties
property.hive.log.dir = /Users/sherlock/devTools/opt/hive-2.3.8/logs
```



> 对了, 最后别忘了把 Mysql 驱动包 copy 到 Hive 的 lib 目录下

### 服务测试

1. 启动 metastore 服务

   ```sh
   hive --service metastore
   ```

2. 启动 hive 客户端

   ```
   hive
   ```

   

