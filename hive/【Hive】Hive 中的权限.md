# 【Hive】Hive 中的权限

本文将介绍一下Hadoop/Hive自带的权限控制，权限控制是大数据平台非常重要的一部分，关乎数据安全。

集群安全下需求：

- 支持多组件，最好能支持当前大数据技术栈的主要组件，HDFS、HBASE、HIVE、YARN、KAFKA等
- 支持细粒度的权限控制，可以达到 HIVE 列，HDFS 目录，HBASE 列,YARN 队列
- 开源，社区活跃，按照现有的集群情况改动尽可能的小，而且要符合业界的趋势。

现有方案：

- Hadoop、Hive 本身的权限控制
- Kerberos安全认证
- Apache Ranger 权限管理方案

## 一、Hadoop 和 Hive 自带的权限

Hadoop权限：

- Hadoop 分布式文件系统实现了一个和 POSIX 系统类似的文件和目录的权限模型
- 每个文件和目录有一个所有者（owner）和一个组（group）
- 文件或目录对其所有者、同组的其他用户以及所有其他用户分别有着不同的权限
- 文件或目录操作都传递路径名给NameNode，对路径做权限检查
- 启动 NameNode 的用户是超级用户，能够通过所有的权限检查
- 通过配置可以指定一组特定的用户为超级用户

Hive 权限

- Hive 可以基于[文件存储](https://cloud.tencent.com/product/cfs?from=10680)级别的权限管理
- Hive 可以基于元数据的权限管理
- User：是基于linux用户的user
- Group：是linux层面上的用户组
- Role：角色在Hive里面创建，给角色添加权限，把角色赋予给 user
- Hive 中没有超级管理员，任何用户都可以进行 Grant/Revoke 操作
- 开发实现自己的权限控制类，确保某个用户为超级用户

------

## 二、3种授权模型

hive 中的用户授权分为两种服务授权: metastore 和 hiveserver2

### 3.1 Storage Based Authorization in the Metastore Server

基于存储的授权 - 可以对Metastore中的元数据进行保护，但是没有提供更加细粒度的访问控制(例如：列级别、行级别)。 

Hive 早期版本是通过 Linux 的用户和用户组来控制用户权限，无法对 Hive 表的 CREATE、SELECT、DROP等操作进行控制。现 Hive 基于元数据库来管理多用户并控制权限。数据分为元数据和数据文件两部分，元数据存储在 mysql，而数据文件则是 HDFS，控制元数据即控制可访问的数据文件。

通过 Hcatcalog API 访问 hive 数据的方式，实际是通过访问 metastore 元数据的形式访问 hive 数据，这类有 MapReduce，Impala，Pig，Spark SQL，Hive Command line等方式，其实就是说这个权限控制发生在和 Metastore 服务交互的时候，实现方式就是在调用 Metastore Api 的时候实现权限校验，主要是防止恶意用户对 Metastore 数据的访问和修改，但是没有提供更加细粒度的访问控制（例如：列级别、行级别）。

下面是其配置

```xml
<property>
  <name>hive.security.metastore.authorization.manager</name>
 <value>org.apache.hadoop.hive.ql.security.authorization.DefaultHiveMetastoreAuthorizationProvider</value>
  <description>authorization manager class name to be used in the metastore for authorization.
  The user defined authorization class should implement interface
  org.apache.hadoop.hive.ql.security.authorization.HiveMetastoreAuthorizationProvider.
  </description>
 </property>
 
<property>
  <name>hive.security.metastore.authenticator.manager</name>
 <value>org.apache.hadoop.hive.ql.security.HadoopDefaultMetastoreAuthenticator</value>
  <description>authenticator manager class name to be used in the metastore for authentication.
  The user defined authenticator should implement interface 
  org.apache.hadoop.hive.ql.security.HiveAuthenticationProvider.
  </description>
</property>
 
<property>
  <name>hive.metastore.pre.event.listeners</name>
  <value> </value>
  <description>pre-event listener classes to be loaded on the metastore side to run code
  whenever databases, tables, and partitions are created, altered, or dropped.
  Set to org.apache.hadoop.hive.ql.security.authorization.AuthorizationPreEventListener
  if metastore-side authorization is desired.
  </description>
</property>
```

### 3.2 SQL Standards Based Authorization in HiveServer2

基于 SQL 标准的 Hive授权。完全兼容 SQL 的授权模型，推荐使用该模式。

除支持对于用户的授权认证，还支持角色 role 的授权认证。role 可理解为是一组权限的集合，通过 role 为用户授权

- 启用当前认证方式之后，dfs, add, delete, compile, reset 等命令被禁用。通过 set 命令设置 hive configuration 的方式被限制某些用户使用。也可通过修改配置文件 hive-site.xml 中hive.security.authorization.sqlstd.confwhitelist 进行配置，哪些用户可以使用这些命令，其实就是白名单（add 或者 drop 函数的命令属于admin 角色所有，所以这个时候如果要添加自定义函数的话可以通过admin 用户添加一个永久函数然后其他用户使用的方式来完成）

- 添加、删除函数以及宏（批量规模）的操作，仅为具有 admin 的用户开放。

- 用户自定义函数（开放支持永久的自定义函数），可通过具有 admin 角色的

- Transform功能被禁用。

> **Role:**
>
> 通过 hiveserver2 的方式访问 hive 数据, 默认提供两种角色：public 和 admin，所有用户默认属于角色 public，而授权则必须是具有 admin 角色的用户才可以完成（普通用户仅可以将自己获得的权限授权给其它用户）
>
> admin 用户可以在配置文件里进行配置，role 的命名是大小写不敏感的，这一点和SQL相同，但是用户名是的大小写敏感的。
>
> ```xml
> <property>
> <name>hive.users.in.admin.role</name>
> <value>root</value>
> </property>
> ```
>
> - public 角色
>
>   默认情况下所有用户默认属于角色 public，而授权则必须是具有角色 admin 的用户才可以完成（普通用户仅可以将自己获得的权限授权给其它用户）
>
>   public 用户有执行授权操作的权限,但是默认情况下 public 用户是没有创建表的权限的
>
> - admin 角色
>
>   用户除了 admin 角色的其他角色都是默认给用户的，也就是说只要你有这个角色的权限，当你执行 `show current roles;`都是可以看到的，但是 admin 角色不行，即使你属于这个角色的列表，你也需要通过`set role admin;` 来获取这个角色的权限
>
> 也就是说 admin 角色是不在用户的 `current roles` 列表里的
>
> 可以通过 `set hive.users.in.admin.role;`查看哪些用户拥有 admin 权限
>
> ```xml
> <!-- 开启身份验证 默认是没有开启的 -->
> <property>
>   <name>hive.security.authorization.enabled</name>
>   <value>true</value>
> </property>
> <property>
>   <name>hive.server2.enable.doAs</name>
>   <value>false</value>
> </property>
> <!-- admin 角色的用户列表  -->
> <property>
>   <name>hive.users.in.admin.role</name>
>   <value>root</value>
> </property>
> <property>
>   <name>hive.security.authorization.manager</name>
>  <value>org.apache.hadoop.hive.ql.security.authorization.plugin.sqlstd.SQLStdHiveAuthorizerFactory</value>
> </property>
> <property>
>   <name>hive.security.authenticator.manager</name>
>  <value>org.apache.hadoop.hive.ql.security.SessionStateUserAuthenticator</value>
> </property>
> 
> <!-- 
> HIVE新建文件权限
> Hive由一个默认的设置来配置新建文件的默认权限。创建文件授权掩码为0002，即664权限，具体要看hadoop与hive用户配置。-->
> <property>  
>   <name>hive.files.umask.value</name>  
>   <value>0002</value>  
>   <description>The dfs.umask value for the hive created folders</description>  
> </property> 
> 
> <!-- 
> HIVE授权存储检查
> 当hive.metastore.authorization.storage.checks属性被设置成true时，Hive将会阻止没有权限的用户进行表删除操作。不过这个配置的默认值是false，应该设置成true。
> -->
> <property>  
>   <name>hive.metastore.authorization.storage.checks</name>  
>   <value>true</value>  
>   <description>Should the metastore do authorization checks against  
>   the underlying storage for operations like drop-partition (disallow  
>   the drop-partition if the user in question doesn't have permissions  
>   to delete the corresponding directory on the storage).</description>  
> </property>
> 
> <!-- 表创建者拥有表的全部权限 
> 这个配置默认是NULL，建议将其设置成ALL，让用户能够访问自己创建的表，否则表的创建者无法访问该表，这明显是不合理的。
> hive 中表是那个用户创建的，那个用户就是Hive 表的Owner 其实就是HDFS 上文件夹的Owner ，这一点和Linux 的操作系统权限是一样的，我用不同的用户登陆进入Hive 终端创建出来的表的Owner所属关系如下图
> -->
> <property>  
>   <name>hive.security.authorization.createtable.owner.grants</name>  
>   <value>ALL</value>  
>   <description>The privileges automatically granted to the owner whenever a table gets created.
>     An example like "select,drop" will grant select and drop privilege to the owner of the table
>   </description>  
> </property>  
> ```
>

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/64766c1363bb4947bf48f43ded156b07~tplv-k3u1fbpfcp-watermark.awebp)

### 3.3 Default Hive Authorization (Legacy Mode)

Hive默认的权限控制并不是完全安全的。hive的权限控制是为了防止用户不小心做了不合适的操作。而不是组织非法用户访问数据

这是因为这种权限管理机制是不完善的，它并没有权限校验的机制，例如你执行`grant` 操作，它并不去检测你是否有权限，它的检测发生在SQL编译的阶段。 


## 三、实操 Hive 的权限操作

在开始之前我们先说一点关于 HiveServer2 的命令行客户端的一个需要注意的事情, 就是当你在不直接指定当前用户的时候登陆上去之后，此时 Hive 的用户并不你系统的当前用户，而是一个匿名用户，我因为这个事情头疼了好一会的，所以开始之前把这个单独说一下

![image-20210112205134932](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/092cdd23f2a04f3c8120690dc7133829~tplv-k3u1fbpfcp-watermark.awebp)

首先添加一个系统用户：

```javascript
[root@hadoop01 ~]# useradd hive
```

将`test`这个表的查询权限赋予给`hive`这个用户：

```javascript
0: jdbc:hive2://localhost:10000> grant select on table test to user hive;
No rows affected (0.12 seconds)
0: jdbc:hive2://localhost:10000> 
```

切换到`hive`用户：

```javascript
[root@hadoop01 ~]# sudo su - hive
```

进入交互命令终端，可以正常执行查询语句：

```javascript
[hive@hadoop01 ~]$ beeline -u jdbc:hive2://localhost:10000 -n hive
...

0: jdbc:hive2://localhost:10000> select user_name from test;
+------------+
| user_name  |
+------------+
| Tom        |
| Jerry      |
| Jim        |
| Angela     |
| Ann        |
| Bella      |
| Bonnie     |
| Caroline   |
+------------+
8 rows selected (0.075 seconds)
0: jdbc:hive2://localhost:10000> 
```

但是如果执行其他操作则会报错提示不支持该操作：

```javascript
0: jdbc:hive2://localhost:10000> delete from test where user_id='f4914b91c5284b01832149776ca53c8d';
Error: Error while compiling statement: FAILED: SemanticException [Error 10294]: Attempt to do update or delete using transaction manager that does not support these operations. (state=42000,code=10294)
0: jdbc:hive2://localhost:10000> 
```

如此一来，我们就可以限制 Hive 中用户对于某些表的操作权限。但之前也提到了，Hive 中没有超级管理员，任何用户都可以进行 Grant/Revoke 操作，这使得权限管理失去意义。为了解决这个问题，就需要我们开发实现自己的权限控制类，确保某个用户为超级用户。

首先创建一个空 Maven 项目，然后添加 `hive-exec` 依赖，完整的 `pom` 文件内容如下：

```javascript
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>hive-security-test</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.apache.hive</groupId>
            <artifactId>hive-exec</artifactId>
            <version>3.1.2</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>8</source>
                    <target>8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

实现自定义权限控制类：

```javascript
package com.example.hive.security;

import com.google.common.base.Joiner;
import org.apache.hadoop.hive.ql.parse.*;
import org.apache.hadoop.hive.ql.session.SessionState;

/**
 * 自定义Hive的超级用户
 *
 * @author 01
 * @date 2020-11-09
 **/
public class HiveAdmin extends AbstractSemanticAnalyzerHook {

    /**
     * 定义超级用户，可以定义多个
     */
    private static final String[] ADMINS = {"root"};

    /**
     * 权限类型列表
     */
    private static final int[] TOKEN_TYPES = {
            HiveParser.TOK_CREATEDATABASE, HiveParser.TOK_DROPDATABASE,
            HiveParser.TOK_CREATEROLE, HiveParser.TOK_DROPROLE,
            HiveParser.TOK_GRANT, HiveParser.TOK_REVOKE,
            HiveParser.TOK_GRANT_ROLE, HiveParser.TOK_REVOKE_ROLE,
            HiveParser.TOK_CREATETABLE
    };

    /**
     * 获取当前登录的用户名
     *
     * @return 用户名
     */
    private String getUserName() {
        boolean hasUserName = SessionState.get() != null &&
                SessionState.get().getAuthenticator().getUserName() != null;

        return hasUserName ? SessionState.get().getAuthenticator().getUserName() : null;
    }

    private boolean isInTokenTypes(int type) {
        for (int tokenType : TOKEN_TYPES) {
            if (tokenType == type) {
                return true;
            }
        }

        return false;
    }

    private boolean isAdmin(String userName) {
        for (String admin : ADMINS) {
            if (admin.equalsIgnoreCase(userName)) {
                return true;
            }
        }

        return false;
    }

    @Override
    public ASTNode preAnalyze(HiveSemanticAnalyzerHookContext context, ASTNode ast) throws SemanticException {
        if (!isInTokenTypes(ast.getToken().getType())) {
            return ast;
        }

        String userName = getUserName();
        if (isAdmin(userName)) {
            return ast;
        }

        throw new SemanticException(userName +
                " is not Admin, except " +
                Joiner.on(",").join(ADMINS)
        );
    }
}
```

将代码打包，上传到服务器上：

```javascript
[root@hadoop01 ~]# ls jars/
hive-security-test-1.0-SNAPSHOT.jar
[root@hadoop01 ~]# 
```

将其拷贝到 Hive 的 `lib` 目录下：

```javascript
[root@hadoop01 ~]# cp jars/hive-security-test-1.0-SNAPSHOT.jar /usr/local/apache-hive-3.1.2-bin/lib/
```

在hive的`hive-site.xml`文件中，增加如下配置：

```javascript
[root@hadoop01 ~]# vim /usr/local/apache-hive-3.1.2-bin/conf/hive-site.xml
<configuration>

    ...

    <property>
        <name>hive.users.in.admin.role</name>
        <value>root</value>
        <description>定义超级管理员，启动的时候会自动创建</description>
    </property>
    <property>
        <name>hive.security.authorization.enabled</name>
        <value>true</value>
        <description>开启权限</description>
    </property>
    <property>
        <name>hive.security.authorization.createtable.owner.grants</name>
        <value>ALL</value>
        <description>表的创建者对表拥有所有权限</description>
    </property>
    <property>
        <name>hive.security.authorization.task.factory</name>
        <value>org.apache.hadoop.hive.ql.parse.authorization.HiveAuthorizationTaskFactoryImpl</value>
        <description>进行权限控制的配置</description>
    </property>
    <property>
        <name>hive.semantic.analyzer.hook</name>
        <value>com.example.hive.security.HiveAdmin</value>
        <description>使用钩子程序，识别超级管理员，进行授权控制</description>
    </property>
</configuration>
```

重启 Hive：

```javascript
[root@hadoop01 ~]# jps
12401 ResourceManager
12898 RunJar
22338 Jps
12500 NodeManager
11948 NameNode
12204 SecondaryNameNode
12047 DataNode
[root@hadoop01 ~]# kill -15 12898
[root@hadoop01 ~]# nohup hiveserver2 -hiveconf hive.execution.engine=mr &
```

进入 Hive 中查看一下角色列表，看看配置是否生效：

```javascript
[root@hadoop01 ~]# beeline -u jdbc:hive2://localhost:10000 -n root
...

0: jdbc:hive2://localhost:10000> set role admin;  # 将当前用户角色设置为admin
No rows affected (0.027 seconds)
0: jdbc:hive2://localhost:10000> show roles;  # 查看角色列表
+---------+
|  role   |
+---------+
| admin   |
| public  |
+---------+
2 rows selected (0.026 seconds)
0: jdbc:hive2://localhost:10000> 
```

测试授权操作：

```javascript
0: jdbc:hive2://localhost:10000> use hive_test;
No rows affected (0.028 seconds)
0: jdbc:hive2://localhost:10000> grant select on table bucket_table to user hive;
No rows affected (0.146 seconds)
0: jdbc:hive2://localhost:10000> 
```

切换到`hive`用户：

```javascript
[root@hadoop01 ~]# sudo su - hive
```

进入交互命令终端，此时执行`grant`语句就会报错，从报错提示可以看到该错误是从我们实现的Hook类里抛出来的：

```javascript
[hive@hadoop01 ~]$ beeline -u jdbc:hive2://localhost:10000 -n hive
...

0: jdbc:hive2://localhost:10000> grant select on table partition_table to user hive;
Error: Error while compiling statement: FAILED: SemanticException hive is not Admin, except root (state=42000,code=40000)
0: jdbc:hive2://localhost:10000> 
```