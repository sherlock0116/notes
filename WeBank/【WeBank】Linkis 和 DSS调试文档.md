# 【WeBank】Linkis 和 DSS调试文档

johnnywang edited this page on 3 Apr 2020 · [2 revisions](https://github.com/WeBankFinTech/Linkis/wiki/Linkis和DSS调试文档/_history)

## 一、前言

  Linkis和DSS的每个微服务都支持调试的，大部分服务都支持本地调试，部分服务只支持远程调试。

1. 支持本地调试的服务

- Eureka：设置的调试Main class是`com.webank.wedatasphere.linkis.eureka.SpringCloudEurekaApplication`
- DSS-Server的主类为：`com.webank.wedatasphere.dss.DSSSpringApplication`
- GateWay/publicservice/metadata/entrance/resourcemanager的Main class都是：`com.webank.wedatasphere.linkis.DataWorkCloudApplication`

2.只支持调试的服务： EngineManager服务以及由EM启动的Engine服务都支持远程调试。

## 二、本地调试服务步骤

  Linkis和DSS的服务都依赖Eureka，所以需要首先启动Eureka服务，Eureka服务也可以用您已经启动的Eureka。Eureka启动后就可以启动其他服务了，启动顺序建议是：GateWay/dss-server/publicservice/metadata/entrance/resourcemanager。

### 2.1 Eureka服务启动

1. 如果不想默认的20303端口可以修改端口配置：

```
文件路径：Linkis\eurekaServer\src\main\resources\application-eureka.yml
修改端口：
server:
  port: 8080 ##启动的端口
```

1. 接着在Idea中新增调试配置 可以通过点击Run或者点击下图的Add Configuration ![01](https://raw.githubusercontent.com/wiki/WeBankFinTech/Linkis/images/debug/01.png)
2. 然后点击新增Application并修改信息 首先设置调试的名字：比如Eureka 接着设置Main class： `com.webank.wedatasphere.linkis.eureka.SpringCloudEurekaApplication` 最后设置该服务的Class Path,对于Eureka的classPath模块是linkis-eureka-server ![02](https://raw.githubusercontent.com/wiki/WeBankFinTech/Linkis/images/debug/02.png)
3. 接着可以点击Debug按钮启动Eureka服务了，并可以通过：http://localhost:8080/访问Eureka页面 ![03](https://raw.githubusercontent.com/wiki/WeBankFinTech/Linkis/images/debug/03.png)

### 2.2 其他服务

1. 需要修改对应服务的Eureka配置，需要修改application.yml文件

```
GateWay：Linkis\gateway\gateway-ujes-support\src\main\resources\application.yml
publicservice:Linkis\publicService\conf\application.yml
metadata:Linkis\metadata\conf\application.yml
entrance:Linkis\ujes\definedEngines\spark\entrance\src\main\resources\application.yml 
Entrance服务有多个需要修改对应目录下面的配置比如spark/hive/python/jdbc等
resourcemanager:Linkis\resourceManager\resourcemanagerserver\src\main\resources\application.yml
DSS-Server:dss-server\src\main\resources\application.yml
```

修改对应的Eureka地址为已经启动的Eureka服务：

```
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8080/eureka/
```

1. 修改linkis和DSS相关的配置,配置文件在linkis.properties，修改对应的配置。
2. 接着新增调试服务 Main Class都统一设置为：`com.webank.wedatasphere.linkis.DataWorkCloudApplication` DSS-Server的为：`com.webank.wedatasphere.dss.DSSSpringApplication` 服务的Class Path为对应的模块：

```
GateWay：linkis-gateway-ujes-support
publicservice:publicservice
metadata:linkis-metadata
entrance:linkis-对应服务名-entrance比如linkis-spark-entrance
resourceManager：linkis-resourcemanager-server
dss-server:dss-server
```

并勾选provide：

![06](https://raw.githubusercontent.com/wiki/WeBankFinTech/Linkis/images/debug/06.png)

1. 接着启动服务，可以看到服务在Eureka页面进行注册：

![05](https://raw.githubusercontent.com/wiki/WeBankFinTech/Linkis/images/debug/05.png)

5.需要注意的两个服务：PublicService和MetaData 因为这两个服务的配置文件在conf目录，需要设置conf目录为resource，如下图 ![04](https://raw.githubusercontent.com/wiki/WeBankFinTech/Linkis/images/debug/04.png) 令外PublicService需要在pom里面加入public-Module模块。 另外resourcemanagerserver的resource有在pom里面exclude，需要去掉排除，或者替换为其他目录

```
<dependency>
 <groupId>com.webank.wedatasphere.linkis</groupId>
 <artifactId>public-module</artifactId>
 <version>${linkis.version}</version>
</dependency>
```

## 三、远程调试服务步骤

  每个服务都支持远程调试，但是需要提前打开远程调试。下面用SparkEngineManager作为调试介绍：

1. 首先修改对应服务bin目录下的start文件添加调试端口：

```
java $SERVER_JAVA_OPTS -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=10092 -cp
```

添加的内容为： `-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=10092` 其中端口可能冲突，可以修改为可用的端口。

1. 接着在idea里面新建一个远程调试，首先选择remote，然后增加服务的host和端口，接着选择调试的模块 ![07](https://raw.githubusercontent.com/wiki/WeBankFinTech/Linkis/images/debug/07.png)
2. 接着点击debug虫就可以完成远程调试了 ![08](https://raw.githubusercontent.com/wiki/WeBankFinTech/Linkis/images/debug/08.png)