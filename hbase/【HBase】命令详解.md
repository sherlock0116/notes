# 【HBase】命令详解

hbase 将命令进行了分组, 例如 ddl, dml, namespace, ......, 进入 hbase shell 之后, 推荐使用 help 命令查看具体 shell 端命令。

```sh
hbase(main):001:0> help
HBase Shell, version 2.2.6, r88c9a386176e2c2b5fd9915d0e9d3ce17d0e456e, Tue Sep 15 17:36:14 CST 2020
Type 'help "COMMAND"', (e.g. 'help "get"' -- the quotes are necessary) for help on a specific command.
Commands are grouped. Type 'help "COMMAND_GROUP"', (e.g. 'help "general"') for help on a command group.

COMMAND GROUPS:
  Group name: general
  Commands: processlist, status, table_help, version, whoami

  Group name: ddl
  Commands: alter, alter_async, alter_status, clone_table_schema, create, describe, disable, disable_all, drop, drop_all, enable, enable_all, exists, get_table, is_disabled, is_enabled, list, list_regions, locate_region, show_filters

  Group name: namespace
  Commands: alter_namespace, create_namespace, describe_namespace, drop_namespace, list_namespace, list_namespace_tables

  Group name: dml
  Commands: append, count, delete, deleteall, get, get_counter, get_splits, incr, put, scan, truncate, truncate_preserve

# 仅展示部分
```

## 一、gerneral

1. processlist

   ```sh
   hbase(main):002:0> processlist
   0 tasks as of: 2021-03-23 22:45:28
   No general tasks currently running.
   Took 6.0665 seconds
   ```

   

2. status

   ```sh
   hbase(main):003:0> status
   1 active master, 0 backup masters, 1 servers, 0 dead, 3.0000 average load
   Took 0.0343 seconds
   ```

   

3. table_help

   ```sh
   hbase(main):004:0> table_help
   Help for table-reference commands.
   
   You can either create a table via 'create' and then manipulate the table via commands like 'put', 'get', etc.
   See the standard help information for how to use each of these commands.
   
   However, as of 0.96, you can also get a reference to a table, on which you can invoke commands.
   For instance, you can get create a table and keep around a reference to it via:
   
      hbase> t = create 't', 'cf'
   
   Or, if you have already created the table, you can get a reference to it:
   
      hbase> t = get_table 't'
   # 仅展示部分
   ```

   

4. whoami

   ```sh
   hbase(main):005:0> whoami
   sherlock (auth:SIMPLE)
       groups: staff, access_bpf, everyone, localaccounts, _appserverusr, admin, _appserveradm, _lpadmin, com.apple.access_screensharing-disabled, com.apple.sharepoint.group.1, _appstore, _lpoperator, _developer, _analyticsusers, com.apple.access_ftp, com.apple.access_ssh, com.apple.access_remote_ae, com.apple.sharepoint.group.4
   Took 0.0721 seconds
   ```

   

5. version

   ```sh
   hbase(main):006:0> version
   2.2.6, r88c9a386176e2c2b5fd9915d0e9d3ce17d0e456e, Tue Sep 15 17:36:14 CST 2020
   Took 0.0004 seconds
   ```

   

## 二、namespace

在 HBase 中，namespace 命名空间指对一组表的逻辑分组，类似RDBMS中的database，方便对表在业务上划分。Apache HBase从0.98.0, 0.95.2两个版本开始支持 namespace 级别的授权操作，HBase 全局管理员可以创建、修改和回收namespace的授权。 
HBase 系统默认定义了两个缺省的 namespace 

- hbase - 系统内建表，包括namespace和meta表 
- default - 用户建表时未指定namespace的表都创建在此 

1. 创建 namespace

   ```sh
   hbase(main):001:0> create_namespace 'sherlock'
   Took 5.8414 seconds
   ```

2. 删除 namespace

   ```sh
   hbase(main):001:0> drop_namespace 'sherlock' 
   Took 5.8414 seconds
   ```

3. 查看 namespace

   ```sh
   hbase(main):007:0> describe_namespace 'sherlock'
   DESCRIPTION
   {NAME => 'sherlock'}
   Quota is disabled
   Took 0.1562 seconds
   ```

4. 列出所有 namespace

   ```sh
   hbase(main):008:0> list_namespace
   NAMESPACE
   default
   hbase
   sherlock
   3 row(s)
   Took 0.0654 seconds
   ```

5. 在 namespace 下创建表

   ```sh
   hbase(main):002:0> create 'sherlock:student', 'person_info'
   Created table sherlock:student
   Took 0.8661 seconds
   => Hbase::Table - sherlock:student
   ```

6. 查看namespace下的表

   ```sh
   hbase(main):004:0> list_namespace_tables 'sherlock'
   TABLE
   student
   1 row(s)
   Took 0.0721 seconds
   => ["student"]
   ```

## 三、Table 

1. HBase 创建表

   ```sh
   hbase(main):002:0> create 'sherlock:student', 'person_info'
   Created table sherlock:student
   Took 0.8661 seconds
   => Hbase::Table - sherlock:student
   ```

2. HBase 列出表

   ```sh
   hbase(main):009:0> list
   TABLE
   sherlock:student
   1 row(s)
   Took 0.0460 seconds
   => ["sherlock:student"]
   ```

   list 是用来列出 HBase 中所有表的命令。如果直接 list，就会把除 hbase 之外的所有的namespace 中的所有表列出；如果想只是列出某个namespace的表，见上面list_namespace_tables。 

3. HBase禁用表

hbase>disable 'person'


4. 查看表是否被禁用

hbase>is_disabled 'person'


5. 禁用所有匹配给定正则表达式的表

hbase>disable_all 'test*'


6. 禁用所有test开头的表 
7. HBase启用表

hbase>enable 'person'


8. 查找表是否被启用

hbase>is_enabled 'person'


9. HBase表描述和修改

hbase> describe 'person'


10. 修改表属性

alter 'person', NAME => 'pengyou1', VERSIONS => 3
修改列pengyou1的VERSIONS属性的值为3。 

11. HBase Exists

hbase>exists 'person'


12. HBase 创建数据 Put

hbase>put 'person','row1','pengyou1:name','zhangsan'
hbase>put 'person','row1','pengyou1:sex','man'
hbase>put 'person','row1','pengyou1:tel','133333333'
hbase>put 'person','row1','pengyou2:name','lisi'
hbase>put 'person','row1','pengyou2:sex','woman'
hbase>put 'person','row1','pengyou2:tel','155555555'


13. HBase更新数据 
    put命令，例如

hbase>put 'person','row1','pengyou1:name','wangwu'
1
HBase读取数据 
get 命令 
读取指定行

hbase>get 'person', 'row1'
1
读取指定列

hbase>get 'person', 'row1','pengyou1:name','pengyou2:name'
1
HBase扫描 
scan命令，类似mysql中的select * from table；

hbase>scan 'person'
1
HBase计数和截断 
可以使用count命令计算表的行数量

hbase>count 'person'
1
truncate此命令将禁止、删除、重新创建一个表。 
这个命令相当于先后执行了disable–>drop–>create命令

hbase>truncate 'person'
1
HBase删除表 
用drop命令可以删除表。在删除一个表之前必须先将其禁用。

hbase>disable  'person'
hbase>drop  'person'
hbase>drop_all  'test*'