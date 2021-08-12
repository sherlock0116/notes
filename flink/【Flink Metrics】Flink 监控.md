# 【Flink Metrics】Flink 监控

下面所有的配置都是在 flink-conf.yaml 中配置

flink 有以下几种 metrics reporter :

## 1、JMX 

JMX (org.apache.flink.metrics.jmx.JMXReporter)

参数：

    port - 连接JMX listens的端口，如果有多个taskmanger在同一台机器，端口可以设置成范围9250-9260

配置：      

```YAML
metrics.reporter.jmx.factory.class: org.apache.flink.metrics.jmx.JMXReporterFactory
metrics.reporter.jmx.port: 8789
```

## 2、Graphite

Graphite (org.apache.flink.metrics.graphite.GraphiteReporter)

复制 \${FLINK_HOME}/plugins/flink-metrics-graphite-1.9.0.jar 到 ${FLINK_HOME}/lib

配置：

````yaml
metrics.reporter.grph.class: org.apache.flink.metrics.graphite.GraphiteReporter
metrics.reporter.grph.host: localhost  # Graphite server host
metrics.reporter.grph.port: 2003   #Graphite server port
metrics.reporter.grph.protocol: TCP  #protocol to use (TCP/UDP)
````

## 3、InfluxDB

InfluxDB (org.apache.flink.metrics.influxdb.InfluxdbReporter)

复制 \${FLINK_HOME}/plugins/flink-metrics-influxdb-1.9.0.jar 到 ${FLINK_HOME}/lib

配置：

```yaml
metrics.reporter.influxdb.class: org.apache.flink.metrics.influxdb.InfluxdbReporter
metrics.reporter.influxdb.host: localhost #the InfluxDB server host
metrics.reporter.influxdb.port: 8086 # (optional) the InfluxDB server port, defaults to 8086
metrics.reporter.influxdb.db: flink #the InfluxDB database to store metrics
metrics.reporter.influxdb.username: flink-metrics # (optional) InfluxDB username
metrics.reporter.influxdb.password: qwerty #(optional) InfluxDB username’s password
metrics.reporter.influxdb.retentionPolicy: one_hour # (optional) InfluxDB retention policy
```

## 4、Prometheus

Prometheus (org.apache.flink.metrics.prometheus.PrometheusReporter)

复制 \${FLINK_HOME}/plugins/flink-metrics-prometheus_2.12-1.12.0.jar 到 ${FLINK_HOME}/lib

配置：端口默认是9249

```yaml
metrics.reporter.prom.class: org.apache.flink.metrics.prometheus.PrometheusReporter
```

## 5、PrometheusPushGateway

PrometheusPushGateway (org.apache.flink.metrics.prometheus.PrometheusPushGatewayReporter)

复制 \${FLINK_HOME}/plugins/flink-metrics-prometheus-1.12.0.jar 到 ${FLINK_HOME}/lib

配置：

```yaml
metrics.reporter.promgateway.class: org.apache.flink.metrics.prometheus.PrometheusPushGatewayReporter
metrics.reporter.promgateway.host: localhost
metrics.reporter.promgateway.port: 9091
metrics.reporter.promgateway.jobName: myJob
metrics.reporter.promgateway.randomJobNameSuffix: true
metrics.reporter.promgateway.deleteOnShutdown: false
```



## 6、StatsD

StatsD (org.apache.flink.metrics.statsd.StatsDReporter)

复制 \${FLINK_HOME}/plugins/flink-metrics-statsd-1.9.0.jar 到 ${FLINK_HOME}/lib

配置：

```yaml
metrics.reporter.stsd.class: org.apache.flink.metrics.statsd.StatsDReporter
metrics.reporter.stsd.host: localhost #the StatsD server host
metrics.reporter.stsd.port: 8125 #the StatsD server port
```



## 7、Datadog

Datadog (org.apache.flink.metrics.datadog.DatadogHttpReporter)

复制 \${FLINK_HOME}/plugins/flink-metrics-datadog-1.9.0.jar 到 ${FLINK_HOME}/lib

配置：

```yaml
metrics.reporter.dghttp.class: org.apache.flink.metrics.datadog.DatadogHttpReporter
metrics.reporter.dghttp.apikey: xxx # the Datadog API key
metrics.reporter.dghttp.tags: myflinkapp,prod #(optional) the global tags that will be applied to metrics when sending to Datadog. Tags should be separated by comma only
metrics.reporter.dghttp.proxyHost: my.web.proxy.com #(optional) The proxy host to use when sending to Datadog
metrics.reporter.dghttp.proxyPort: 8080 #(optional) The proxy port to use when sending to Datadog, defaults to 8080
```



## 8、Slf4j

Slf4j (org.apache.flink.metrics.slf4j.Slf4jReporter)

复制 \${FLINK_HOME}/plugins/flink-metrics-slf4j-1.9.0.jar 到 ${FLINK_HOME}/lib

配置：

```yaml
metrics.reporter.slf4j.class: org.apache.flink.metrics.slf4j.Slf4jReporter
metrics.reporter.slf4j.interval: 60 SECONDS
```

