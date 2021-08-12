# 【Hive】安装使用过程中遇到的坑

## 1. Column length too big for column 'PARAM_VALUE' (max = 21845)

这是在创建 hive 表的时候遇到的坑. 大致原因是因为mysql是使用utfmb4编码，导致该字段在编码的时候内容过长（gbk使用双字节，utf使用三字节）

### 解决方案

因为这是mysql编码格式的问题,
进入mysql,输入这三条命令:
`show variables like "char%";`
`use <hive元数据库>;`
`alter database metastore character set latin1;`

修改前

![修改前](/Users/sherlock/Desktop/notes/allPics/Hive/修改前.png)

修改后

![修改后](/Users/sherlock/Desktop/notes/allPics/Hive/修改后.png)



## 2.beeline连接时发生的权限问题

这个大概报错如下

```
Error: Could not open client transport with JDBC Uri: jdbc:hive2://s1:10000/hive: Failed to open new session: java.lang.RuntimeException: org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.security.authorize.AuthorizationException): User: xxx  is not allowed to impersonate anonymous (state=08S01,code=0)
```

### 解决方式

在hadoop的配置文件core-site.xml增加如下配置, 其中“xxx”是连接beeline的用户，将“xxx”替换成自己的用户名即可

```html
<property>
	<name>hadoop.proxyuser.xxx.hosts</name>
	<value>*</value>
</property>
<property>
  <name>hadoop.proxyuser.xxx.groups</name>
  <value>*</value>
</property>
```

之后刷新配置

```sh
➜  hive-2.3.7 hadoop dfsadmin -fs hdfs://master-IP:9000 -refreshSuperUserGroupsConfiguration
```

