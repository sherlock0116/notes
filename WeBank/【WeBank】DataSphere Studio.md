# 【WeBank】DataSphere Studio

DataSphere Studio（简称DSS）是微众银行自研的一站式数据应用开发管理门户。

基于插拔式的集成框架设计，及计算中间件 [**Linkis**](https://github.com/WeBankFinTech/Linkis) ，可轻松接入上层各种数据应用系统，让数据开发变得简洁又易用。

在统一的 UI 下，DataSphere Studio 以工作流式的图形化拖拽开发体验，将满足从数据交换、脱敏清洗、分析挖掘、质量检测、可视化展现、定时调度到数据输出应用等，数据应用开发全流程场景需求。

**DSS通过插拔式的集成框架设计，让用户可以根据需要，简单快速替换DSS已集成的各种功能组件，或新增功能组件。**

借助于 [**Linkis**](https://github.com/WeBankFinTech/Linkis) 计算中间件的连接、复用与简化能力，DSS天生便具备了金融级高并发、高可用、多租户隔离和资源管控等执行与调度能力。

## 一、界面预览

![11](https://github.com/sherlock0116/DataSphereStudio/raw/dev-0.9.1/images/en_US/readme/DSS_gif.gif)

## 二、核心特点

### 2.1 DSS 主要特点：

#### 2.1.1 一站式、全流程的应用开发管理界面

DSS 集成度极高，目前已集成的系统有：

1. 数据开发IDE工具——[Scriptis](https://github.com/WeBankFinTech/Scriptis)
2. 数据可视化工具——[Visualis](https://github.com/WeBankFinTech/Visualis)（基于宜信[Davinci](https://github.com/edp963/davinci)二次开发）
3. 数据质量管理工具——[Qualitis](https://github.com/WeBankFinTech/Qualitis)
4. 工作流调度工具——[Azkaban](https://azkaban.github.io/)

​    **DSS 插拔式的框架设计模式，允许用户快速替换 DSS 已集成的各个 Web 系统**。如：将 Scriptis 替换成Zeppelin，将 Azkaban 替换成 DolphinScheduler。

![](https://github.com/sherlock0116/DataSphereStudio/raw/dev-0.9.1/images/zh_CN/readme/onestop.gif)

#### 2.1.2 基于 Linkis 计算中间件，打造独有的 AppJoint 设计理念

​    AppJoint，是DSS可以简单快速集成各种上层Web系统的核心概念。

​    AppJoint——应用关节，定义了一套统一的前后台接入规范，可让外部数据应用系统快速简单地接入，成为DSS数据应用开发中的一环。

​    DSS通过串联多个AppJoint，编排成一条支持实时执行和定时调度的工作流，用户只需简单拖拽即可完成数据应用的全流程开发。

​    由于AppJoint对接了Linkis，外部数据应用系统因此具备了资源管控、并发限流、用户资源管理等能力，且允许上下文信息跨系统级共享，彻底告别应用孤岛。

#### 2.1.3 Project 级管理单元

​    以Project为管理单元，组织和管理各数据应用系统的业务应用，定义了一套跨数据应用系统的项目协同开发通用标准。

## 三、已集成的数据应用组件

DSS 通过实现多个 AppJoint，已集成了丰富多样的各种上层数据应用系统，基本可满足用户的数据开发需求。

**用户如果有需要，也可以轻松集成新的数据应用系统，以替换或丰富DSS的数据应用开发流程。**

### 3.1 DSS的调度能力——Azkaban AppJoint

用户的很多数据应用，通常希望具备周期性的调度能力。

目前市面上已有的开源调度系统，与上层的其他数据应用系统整合度低，且难以融通。

DSS 通过实现 Azkaban AppJoint，允许用户将一个编排好的工作流，一键发布到 Azkaban 中进行定时调度。

DSS 还为调度系统定义了一套标准且通用的 DSS 工作流解析发布规范，让其他调度系统可以轻松与 DSS 实现低成本对接。

![](https://github.com/sherlock0116/DataSphereStudio/raw/dev-0.9.1/images/zh_CN/readme/Azkaban_AppJoint.gif)

### 3.2 数据开发——Scriptis AppJoint

什么是[Scriptis](https://github.com/WeBankFinTech/Scriptis)?

​	Scriptis是一款支持在线写SQL、Pyspark、HiveQL等脚本，提交给[Linkis](https://github.com/WeBankFinTech/Linkis)执行的数据分析Web工具，且支持UDF、函数、资源管控和智能诊断等企业级特性。

​	Scriptis AppJoint为DSS集成了Scriptis的数据开发能力，并允许Scriptis的各种脚本类型，作为DSS工作流的节点，参与到应用开发的流程中。

​	目前已支持HiveSQL、SparkSQL、Pyspark、Scala等脚本节点类型。

![](https://github.com/sherlock0116/DataSphereStudio/raw/dev-0.9.1/images/zh_CN/readme/Scriptis_AppJoint.gif)

### 3.3 数据可视化——Visualis AppJoint

什么是 Visualis?

​	Visualis 是一个基于宜信开源项目 Davinci 二次开发的数据可视化 BI 工具，为用户在数据安全和权限方面，提供金融级数据可视化能力。

​	Visualis AppJoint 为 DSS 集成了 Visualis 的数据可视化能力，并允许数据大屏和仪表盘，作为 DSS工作流的节点，与上游的数据集市关联起来。

![](https://github.com/sherlock0116/DataSphereStudio/raw/dev-0.9.1/images/zh_CN/readme/Visualis_AppJoint.gif)

### 3.4 数据质量——Qualitis AppJoint

​	Qualitis AppJoint 为 DSS 集成数据质量校验能力，将数据质量系统集成到 DSS 工作流开发中，对数据完整性、正确性等进行校验。

![](https://github.com/sherlock0116/DataSphereStudio/raw/dev-0.9.1/images/zh_CN/readme/Qualitis_AppJoint.gif)

### 3.5 数据发送——Sender AppJoint

​	Sender AppJoint 为 DSS 集成数据发送能力，目前支持 SendEmail 节点类型，所有其他节点的结果集，都可以通过邮件发送。

​	例如：SendEmail 节点可直接将 Display 数据大屏作为邮件发送出来。

### 3.6 信号节点——Signal AppJoint

​	EventChecker AppJoint用于强化业务与流程之间的解耦和相互关联。

​	DataChecker节点：检查库表分区是否存在。

​	EventSender: 跨工作流和工程的消息发送节点。

​	EventReceiver: 跨工作流和工程的消息接收节点。

### 3.7 功能节点

​	空节点、子工作流节点。

### 3.8 节点扩展

​	 **根据需要，用户可以简单快速替换DSS已集成的各种功能组件，或新增功能组件。**

## 四、使用场景

DataSphere Studio 适用于以下场景：

1. 正在筹建或初步具备大数据平台能力，但无任何数据应用工具的场景。
2. 已具备大数据基础平台能力，且仅有少数数据应用工具的场景。
3. 已具备大数据基础平台能力，且拥有全部数据应用工具，但工具间尚未打通，用户使用隔离感强、学习成本高的场景。
4. 已具备大数据基础平台能力，且拥有全部数据应用工具，部分工具已实现对接，但尚未定义统一规范的场景。

## 五、架构

![](https://github.com/sherlock0116/DataSphereStudio/raw/dev-0.9.1/images/zh_CN/readme/architecture.png)

