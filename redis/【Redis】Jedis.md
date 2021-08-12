# 【Redis】Jedis

### 概述

[Jedis](https://link.jianshu.com/?t=https://github.com/xetorthio/jedis)是Redis官方推荐的Java连接开发工具。要在 Java 开发中使用好 Redis 中间件，必须对 Jedis 熟悉才能写成漂亮的代码。这篇文章不描述怎么安装 Redis 和 Reids 的命令，只对 Jedis 的使用进行对介绍。

![img](https:////upload-images.jianshu.io/upload_images/3167863-5e6eec8d6347bce5.png?imageMogr2/auto-orient/strip|imageView2/2/w/308)



## 1. 基本使用

Jedis 的基本使用非常简单，只需要创建 Jedis 对象的时候指定 host，port, password 即可。当然，Jedis 对象又很多构造方法，都大同小异，只是对应和Redis连接的socket的参数不一样而已。

```csharp
Jedis jedis = new Jedis("localhost", 6379);  //指定Redis服务Host和port
jedis.auth("xxxx"); //如果Redis服务连接需要密码，制定密码
String value = jedis.get("key"); //访问Redis服务
jedis.close(); //使用完关闭连接
```

Jedis基本使用十分简单，在每次使用时，构建Jedis对象即可。在Jedis对象构建好之后，Jedis底层会打开一条Socket通道和Redis服务进行连接。所以在使用完Jedis对象之后，需要调用Jedis.close()方法把连接关闭，不如会占用系统资源。当然，如果应用非常平凡的创建和销毁Jedis对象，对应用的性能是很大影响的，因为构建Socket的通道是很耗时的(类似数据库连接)。我们应该使用连接池来减少Socket对象的创建和销毁过程。

## 2. 连接池使用

Jedis连接池是基于apache-commons pool2实现的。在构建连接池对象的时候，需要提供池对象的配置对象，及JedisPoolConfig(继承自GenericObjectPoolConfig)。我们可以通过这个配置对象对连接池进行相关参数的配置(如最大连接数，最大空数等)。

```csharp
JedisPoolConfig config = new JedisPoolConfig();
config.setMaxIdle(8);
config.setMaxTotal(18);
JedisPool pool = new JedisPool(config, "127.0.0.1", 6379, 2000, "password");
Jedis jedis = pool.getResource();
String value = jedis.get("key");
......
jedis.close();
pool.close();
```

### 2.1 JedisPoolConfig

`JedisPoolConfig config = new JedisPoolConfig();`

>1. config.setBlockWhenExhausted(``true``);
>
>   连接耗尽时是否阻塞, false报异常, ture阻塞直到超时, 默认true
>
>2. config.setEvictionPolicyClassName(``"org.apache.commons.pool2.impl.DefaultEvictionPolicy"``);
>
>   设置的逐出策略类名, 默认 DefaultEvictionPolicy(当连接超过最大空闲时间,或连接数超过最大空闲连接数)
>
>4. 是否启用pool的jmx管理功能, 默认true
>     config.setJmxEnabled(``true``);
>
>5. MBean oName = new ObjectName("org.apache.commons.pool2:type=GenericObjectPool,name=" + "pool" + i); 默认为"pool", JMX不熟, 具体不知道是干啥的...默认就好.
>     config.setJmxNamePrefix(``"pool"``);
>
>6. 是否启用后进先出, 默认true
>     config.setLifo(``true``);
>7. 最大空闲连接数, 默认8个
>     config.setMaxIdle(``8``);
>8. 获取连接时的最大等待毫秒数(如果设置为阻塞时BlockWhenExhausted),如果超时就抛异常, 小于零:阻塞不确定的时间, 默认-1
>     config.setMaxWaitMillis(-``1``);
>9. 逐出连接的最小空闲时间 默认1800000毫秒(30分钟)
>     config.setMinEvictableIdleTimeMillis(``1800000``);
>10. 最小空闲连接数, 默认0
>       config.setMinIdle(``0``);
>11. 每次逐出检查时 逐出的最大数目 如果为负数就是 : 1/abs(n), 默认3
>       config.setNumTestsPerEvictionRun(``3``);
>12. 对象空闲多久后逐出, 当空闲时间 > 该值且空闲连接>最大空闲数时直接逐出, 不再根据MinEvictableIdleTimeMillis判断 (默认逐出策略)  
>       config.setSoftMinEvictableIdleTimeMillis(``1800000``);
>13. 在获取连接的时候检查有效性, 默认false
>       config.setTestOnBorrow(``false``);
>14. 逐出扫描的时间间隔(毫秒) 如果为负数,则不运行逐出线程, 默认-1
>       config.setTimeBetweenEvictionRunsMillis(-``1``);
>
>```java
>JedisPool pool = new JedisPool(config, "localhost", );
>int timeout=3000;
>new JedisSentinelPool(master, sentinels, poolConfig,timeout);
>//timeout 读取超时
>```

使用Jedis连接池之后，在每次用完连接对象后一定要记得把连接归还给连接池。Jedis对close方法进行了改造，如果是连接池中的连接对象，调用Close方法将会是把连接对象返回到对象池，若不是则关闭连接。可以查看如下代码

```kotlin
@Override
public void close() { //Jedis的close方法
    if (dataSource != null) {
        if (client.isBroken()) {
            this.dataSource.returnBrokenResource(this);
        } else {
            this.dataSource.returnResource(this);
        }
    } else {
        client.close();
    }
}

//另外从对象池中获取Jedis链接时，将会对dataSource进行设置
// JedisPool.getResource()方法
public Jedis getResource() {
    Jedis jedis = super.getResource();   
    jedis.setDataSource(this);
    return jedis;
}
```

## 3. 高可用连接

我们知道，连接池可以大大提高应用访问Reids服务的性能，减去大量的Socket的创建和销毁过程。但是Redis为了保障高可用，服务一般都是Sentinel部署方式([可以查看我的文章详细了解](https://www.jianshu.com/p/cbd40a188226))。当Redis服务中的主服务挂掉之后，会仲裁出另外一台Slaves服务充当Master。这个时候，我们的应用即使使用了Jedis连接池，Master服务挂了，我们的应用奖还是无法连接新的Master服务。为了解决这个问题，Jedis也提供了相应的Sentinel实现，能够在Redis Sentinel主从切换时候，通知我们的应用，把我们的应用连接到新的 Master服务。先看下怎么使用。

> 注意：Jedis版本必须2.4.2或更新版本



```csharp
Set<String> sentinels = new HashSet<>();
sentinels.add("172.18.18.207:26379");
sentinels.add("172.18.18.208:26379");
JedisPoolConfig config = new JedisPoolConfig();
config.setMaxIdle(5);
config.setMaxTotal(20);
JedisSentinelPool pool = new JedisSentinelPool("mymaster", sentinels, config);
Jedis jedis = pool.getResource();
jedis.set("jedis", "jedis");
......
jedis.close();
pool.close();
```

Jedis Sentinel的使用也是十分简单的，只是在JedisPool中添加了Sentinel和MasterName参数。Jedis Sentinel底层基于Redis订阅实现Redis主从服务的切换通知。当Reids发生主从切换时，Sentinel会发送通知主动通知Jedis进行连接的切换。JedisSentinelPool在每次从连接池中获取链接对象的时候，都要对连接对象进行检测，如果此链接和Sentinel的Master服务连接参数不一致，则会关闭此连接，重新获取新的Jedis连接对象。



```java
public Jedis getResource() {
    while (true) {
        Jedis jedis = super.getResource();
        jedis.setDataSource(this);

        // get a reference because it can change concurrently
        final HostAndPort master = currentHostMaster;
        final HostAndPort connection = new HostAndPort(jedis.getClient().getHost(), jedis.getClient().getPort());
        if (master.equals(connection)) {
            // connected to the correct master
            return jedis;
        } else {
            returnBrokenResource(jedis);
        }
    }
}
```

当然，JedisSentinelPool对象要时时监控RedisSentinel的主从切换。在其内部通过Reids的订阅实现。具体的实现看JedisSentinelPool的两个方法就很清晰

```dart
private HostAndPort initSentinels(Set<String> sentinels, final String masterName) {
    HostAndPort master = null;
    boolean sentinelAvailable = false;
    log.info("Trying to find master from available Sentinels...");
    for (String sentinel : sentinels) {
        final HostAndPort hap = HostAndPort.parseString(sentinel);
        log.fine("Connecting to Sentinel " + hap);
        Jedis jedis = null;
        try {
            jedis = new Jedis(hap.getHost(), hap.getPort());
            //从RedisSentinel中获取Master信息
            List<String> masterAddr = jedis.sentinelGetMasterAddrByName(masterName);
            sentinelAvailable = true; // connected to sentinel...
            if (masterAddr == null || masterAddr.size() != 2) {
                log.warning("Can not get master addr, master name: " + masterName + ". Sentinel: " + hap + ".");
                continue;
            }
            master = toHostAndPort(masterAddr);
            log.fine("Found Redis master at " + master);
            break;
        } catch (JedisException e) {
            // it should handle JedisException there's another chance of raising JedisDataException
            log.warning("Cannot get master address from sentinel running @ " + hap + ". Reason: " + e + ". Trying next one.");
        } finally {
            if (jedis != null) {
                jedis.close();
            }
        }
    }
    if (master == null) {
        if (sentinelAvailable) {
            // can connect to sentinel, but master name seems to not monitored
            throw new JedisException("Can connect to sentinel, but " + masterName + " seems to be not monitored...");
        } else {
            throw new JedisConnectionException("All sentinels down, cannot determine where is " + masterName + " master is running...");
        }
    }
    log.info("Redis master running at " + master + ", starting Sentinel listeners...");
    //启动后台线程监控RedisSentinal的主从切换通知
    for (String sentinel : sentinels) {
        final HostAndPort hap = HostAndPort.parseString(sentinel);
        MasterListener masterListener = new MasterListener(masterName, hap.getHost(), hap.getPort());
        // whether MasterListener threads are alive or not, process can be stopped
        masterListener.setDaemon(true);
        masterListeners.add(masterListener);
        masterListener.start();
    }
    return master;
}


private void initPool(HostAndPort master) {
    if (!master.equals(currentHostMaster)) {
        currentHostMaster = master;
        if (factory == null) {
            factory = new JedisFactory(master.getHost(), master.getPort(), connectionTimeout, soTimeout, password, database, clientName, false, null, null, null);
            initPool(poolConfig, factory);
        } else {
            factory.setHostAndPort(currentHostMaster);
            // although we clear the pool, we still have to check the returned object
            // in getResource, this call only clears idle instances, not
            // borrowed instances
            internalPool.clear();
        }
        log.info("Created JedisPool to master at " + master);
    }
}
```

可以看到，JedisSentinel的监控时使用MasterListener这个对象来实现的。看对应源码可以发现是基于Redis的订阅实现的，其订阅频道为"+switch-master"。当MasterListener接收到switch-master消息时候，会使用新的Host和port进行initPool。这样对连接池中的连接对象清除，重新创建新的连接指向新的Master服务。

## 4. 客户端分片

对于大应用来说，单台Redis服务器肯定满足不了应用的需求。在Redis3.0之前，是不支持集群的。如果要使用多台Reids服务器，必须采用其他方式。很多公司使用了代理方式来解决Redis集群。对于Jedis，也提供了客户端分片的模式来连接“Redis集群”。其内部是采用Key的一致性hash算法来区分key存储在哪个Redis实例上的。



```csharp
JedisPoolConfig config = new JedisPoolConfig();
config.setMaxTotal(500);
config.setTestOnBorrow(true);
List<JedisShardInfo> jdsInfoList = new ArrayList<>(2);
jdsInfoList.add(new JedisShardInfo("192.168.2.128", 6379));
jdsInfoList.add(new JedisShardInfo("192.168.2.108", 6379));
pool = new ShardedJedisPool(config, jdsInfoList, Hashing.MURMUR_HASH, Sharded.DEFAULT_KEY_TAG_PATTERN);
jds.set(key, value);
......
jds.close();
pool.close();
```

当然，采用这种方式也存在两个问题

1. 扩容问题：
   因为使用了一致性哈稀进行分片，那么不同的key分布到不同的Redis-Server上，当我们需要扩容时，需要增加机器到分片列表中，这时候会使得同样的key算出来落到跟原来不同的机器上，这样如果要取某一个值，会出现取不到的情况。
2. 单点故障问题：
   当集群中的某一台服务挂掉之后，客户端在根据一致性hash无法从这台服务器取数据。

对于扩容问题，Redis的作者提出了一种名为Pre-Sharding的方式。即事先部署足够多的Redis服务。
对于单点故障问题，我们可以使用Redis的HA高可用来实现。利用Redis-Sentinal来通知主从服务的切换。当然，Jedis没有实现这块。我将会在下一篇文章进行介绍。

## 5. 小结

> 对于Jedis的基本使用还是很简单的。要根据不用的应用场景选择对于的使用方式。
> 另外，Spring也提供了Spring-data-redis包来整合Jedis的操作，另外Spring也单独分装了Jedis(我将会在另外一篇文章介绍)。
