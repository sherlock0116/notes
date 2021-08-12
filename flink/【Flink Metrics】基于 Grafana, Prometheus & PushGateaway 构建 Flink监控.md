# 【Flink Metrics】基于 Grafana, Prometheus & PushGateaway 构建 Flink监控

![](https://img2018.cnblogs.com/blog/758511/201909/758511-20190902100620781-172520634.png)

Flink App ： 通过report 将数据发出去

Pushgateway : Prometheus 生态中一个重要工具

Prometheus : 一套开源的系统监控报警框架 （[Prometheus 入门与实践](https://www.ibm.com/developerworks/cn/cloud/library/cl-lo-prometheus-getting-started-and-practice/index.html)）

Grafana： 一个跨平台的开源的度量分析和可视化工具，可以通过将采集的数据查询然后可视化的展示，并及时通知（[可视化工具Grafana：简介及安装](https://www.cnblogs.com/imyalost/p/9873641.html)）

Node_exporter : 跟Pushgateway一样是Prometheus 的组件，采集到主机的运行指标如CPU, 内存，磁盘等信息

以下安装，大部分参考博客： https://www.cnblogs.com/xiao987334176/p/9930517.html#autoid-0-0-0

**1、docker pull 镜像**

```
docker pull prom/node-exporter
docker pull prom/pushgateway
docker pull prom/prometheus
docker pull grafana/grafana
```

查看下载的镜像

```
$ docker images
REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
prom/prometheus      latest              d5b9d7ed160a        2 weeks ago         138MB
grafana/grafana      latest              a6e14b4109af        2 weeks ago         253MB
prom/pushgateway     latest              20e6dcae675f        4 weeks ago         19.2MB
prom/node-exporter   latest              e5a616e4b9cf        2 months ago        22.9MB
```

**2、编辑prometheus.yml 、创建 Grafana 数据存储目录**

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
$ mkdir /opt/grafana-storage  # grafana 数据存储目录

$ cat /opt/prometheus/prometheus.yml # prometheus 配置
global:
  scrape_interval:     60s
  evaluation_interval: 60s
 
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']
        labels:
          instance: prometheus
 
  - job_name: linux
    static_configs:
      - targets: ['venn:9100']
        labels:
          instance: localhost
  - job_name: 'pushgateway'
    static_configs:
      - targets: ['venn:9091']
        labels:
          instance: 'pushgateway'
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

**3、启动各个组件**

```
docker run -d -p 3000:3000   --name=grafana   -v /opt/grafana-storage:/var/lib/grafana   grafana/grafana
docker run -d -p 9100:9100  -v "/proc:/host/proc:ro"  -v "/sys:/host/sys:ro"  -v "/:/rootfs:ro"  --net="host"  prom/node-exporter
docker run -d -p 9090:9090  -v /opt/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml  prom/prometheus
docker run -d -p 9091:9091 prom/pushgateway
```

查看docker进程

```
$ docker ps
CONTAINER ID        IMAGE                COMMAND                  CREATED             STATUS              PORTS                    NAMES
4a689cf48e10        prom/pushgateway     "/bin/pushgateway"       5 days ago          Up 5 days           0.0.0.0:9091->9091/tcp   infallible_goldstine
fcc40433bf75        grafana/grafana      "/run.sh"                5 days ago          Up 5 days           0.0.0.0:3000->3000/tcp   grafana
8ba942d0cf35        prom/prometheus      "/bin/prometheus --c…"   5 days ago          Up 5 days           0.0.0.0:9090->9090/tcp   quizzical_colden
b84b0f4be2b2        prom/node-exporter   "/bin/node_exporter"     5 days ago          Up 5 days                                    fervent_poitras
```

查看端口

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
$ netstat -apn | grep -E '9091|3000|9090|9100'
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 172.17.0.1:39028        172.17.0.4:9091         ESTABLISHED -                   
tcp6       0      0 :::9100                 :::*                    LISTEN      -                   
tcp6       0      0 :::3000                 :::*                    LISTEN      -                   
tcp6       0      0 :::9090                 :::*                    LISTEN      -                   
tcp6       0      0 :::9091                 :::*                    LISTEN      -                   
tcp6       0      0 192.168.229.129:45864   192.168.229.128:9091    TIME_WAIT   -                   
tcp6       0      0 192.168.229.129:45856   192.168.229.128:9091    TIME_WAIT   -                   
tcp6       0      0 192.168.229.129:45824   192.168.229.128:9091    TIME_WAIT   -                   
tcp6       0      0 192.168.229.129:45874   192.168.229.128:9091    TIME_WAIT   -                   
tcp6       0      0 192.168.229.129:45854   192.168.229.128:9091    TIME_WAIT   -                   
tcp6       0      0 192.168.229.129:45836   192.168.229.128:9091    TIME_WAIT   -                   
tcp6       0      0 192.168.229.129:45814   192.168.229.128:9091    TIME_WAIT   -                   
tcp6       0      0 192.168.229.128:9100    192.168.229.1:13405     ESTABLISHED -                   
tcp6       0      0 192.168.229.129:45826   192.168.229.128:9091    TIME_WAIT   -                   
tcp6       0      0 192.168.229.129:45844   192.168.229.128:9091    TIME_WAIT   -                   
tcp6       0      0 192.168.229.128:9091    172.17.0.2:53930        ESTABLISHED -                   
tcp6       0      0 192.168.229.129:45846   192.168.229.128:9091    TIME_WAIT   -                   
tcp6       0      0 192.168.229.128:9100    172.17.0.2:54776        ESTABLISHED -                   
tcp6       0      0 192.168.229.129:45816   192.168.229.128:9091    TIME_WAIT   -                   
tcp6       0      0 192.168.229.129:45876   192.168.229.128:9091    ESTABLISHED 40846/java          
tcp6       0      0 192.168.229.129:45834   192.168.229.128:9091    TIME_WAIT   -                   
tcp6       0      0 192.168.229.129:45866   192.168.229.128:9091    TIME_WAIT   -   
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

**4、查看组件页面**

```
node_exporter:  ip:9100/metrics
```

![img](https://img2018.cnblogs.com/blog/758511/201909/758511-20190902104205571-781738551.png)

 

**查看 prometheus： ip:9090/targets**

![img](https://img2018.cnblogs.com/blog/758511/201909/758511-20190902104435357-2072450978.png)

如果state 不是 UP 的，等一会就起来了 

**查看Grafana：** 

 ![img](https://img2018.cnblogs.com/blog/758511/201909/758511-20190902104824304-1930075361.png)

 默认用户名密码 ： amin/admin

此处不再赘述，**配置数据源、创建系统负载监控**参考博客：https://www.cnblogs.com/xiao987334176/p/9930517.html#autoid-0-0-0 

**5、配置Flink report ：**

在Flink 配置文件 flink-conf.yml 中添加如下内容：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
##metrics
metrics.reporter.promgateway.class: org.apache.flink.metrics.prometheus.PrometheusPushGatewayReporter
metrics.reporter.promgateway.host: venn
metrics.reporter.promgateway.port: 9091
metrics.reporter.promgateway.jobName: myJob
metrics.reporter.promgateway.randomJobNameSuffix: true
metrics.reporter.promgateway.deleteOnShutdown: false
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

启动一个任务（上一篇博客的案例迟到数据处理）：

```
flink run -m yarn-cluster -ynm LateDataProcess -yn 1 -c com.venn.stream.api.sideoutput.lateDataProcess.LateDataProcess jar/flinkDemo-1.0.jar
```

查看任务webUI：

![img](https://img2018.cnblogs.com/blog/758511/201909/758511-20190902105953007-435149092.png)

PS：任务已经跑了一段时间了

**6、Grafana 中配置Flink监控**

由于上面一句配置好Flink report、 pushgateway、prometheus，并且在Grafana中已经添加了prometheus 数据源，所以Grafana中会自动获取到 flink job的metrics 。

 Grafana 首页，点击New dashboard，创建一个新的dashboard

![img](https://img2018.cnblogs.com/blog/758511/201909/758511-20190902110450070-679198098.png)

选中之后，即会出现对应的监控指标

![img](https://img2018.cnblogs.com/blog/758511/201909/758511-20190902110847297-1453501511.png)

至此，Flink 的metrics 的指标展示在Grafana 中了

flink 指标对应的指标名比较长，可以在Legend 中配置显示内容，在{{key}} 将key换成对应需要展示的字段即可，如： {{job_name}},{{operator_name}}

对应显示如下：

![img](https://img2018.cnblogs.com/blog/758511/201909/758511-20190902111630764-713960498.png)