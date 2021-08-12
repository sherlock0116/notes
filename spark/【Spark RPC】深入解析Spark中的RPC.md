# 【Spark RPC】深入解析Spark中的RPC

Spark是一个快速的、通用的分布式计算系统，而分布式的特性就意味着，必然存在节点间的通信。本文主要介绍不同的Spark组件之间是如何通过RPC（Remote Procedure Call) 进行点对点通信的，分为三个章节：

- Spark RPC的简单示例和实际应用；
- Spark RPC模块的设计原理；
- Spark RPC核心技术总结。

## 一、Spark RPC的简单示例和实际应用

Spark 的 RPC 主要在两个模块中：

- 在 Spark-core 中，主要承载了更好的封装 server 和 client 的作用，以及和 scala 语言的融合，它依赖于模块 org.apache.spark.spark-network-common；
- 在 org.apache.spark.spark-network-common 中，该模块是 java 语言编写的，最新版本是基于netty4 开发的，提供全双工、多路复用 I/O 模型的 Socket I/O 能力，Spark 的传输协议结构（wire protocol）也是自定义的。

为了更好的了解 Spark RPC 的内部实现细节，我基于 Spark 2.1 版本抽离了 RPC 通信的部分，单独启了一个[项目](https://github.com/neoremind/kraps-rpc)，放到了 github 以及发布到 Maven 中央仓库做学习使用，提供了比较好的上手文档、参数设置和性能评估。下面就通过这个模块对 Spark RPC 先做一个感性的认识。

以下的代码均可以在[kraps-rpc](http://link.zhihu.com/?target=https%3A//github.com/neoremind/kraps-rpc)找到。

### 1.1 简单示例

假设我们要开发一个 Hello 服务，客户端可以传输 string，服务端响应 hi 或者 bye，并 echo 回去输入的string。

 **第一步**，定义一个 HelloEndpoint 继承自 RpcEndpoint 表明可以并发的调用该服务，如果继承自ThreadSafeRpcEndpoint 则表明该 Endpoint 不允许并发。