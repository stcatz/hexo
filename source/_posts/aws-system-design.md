---
title: '[译]如何设计一个在AWS上可以扩展到数百万用户的系统'
date: 2017-04-01 17:00:58
tags: ['架构', '面试']
---

{% blockquote The System Design Primer https://github.com/donnemartin/system-design-primer Design a system that scales to millions of users on AWS %}
译注：《The System Design Primer》是一位大神[Donne Martin](http://donnemartin.com/)写的关于如何回答面试中系统设计方面问题的电子书，其中《如何设计一个在AWS上可以扩展到数百万用户的系统》最具有代表性。我觉得这篇文章不仅可以用来准备面试，更是可以作为开始一个新项目的架构指南。当然一篇文章能容纳的内容有限，本篇所涉及的很多话题可以很轻易的扩展成一本书，很多大公司会有专门的团队负责开发/维护文中的一些模块。读者可以把本篇文章作为一个互联网系统架构设计的提纲，在有需要的话题下再深入下去。
{% endblockquote %}

<!-- more -->

# 设计一个在AWS上可以扩展到数百万用户的系统
*注意：为避免重复，本文档中有些话题直接链接到[系统设计主题](https://github.com/donnemartin/system-design-primer#index-of-system-design-topics)中的相关区域的。请参阅链接内容，了解一般回答要点，取舍和替代方案。*
## 步骤1：用例和约束条件
>明确问题的需求和适用范围。
>提出问题来澄清用例和约束条件。
>讨论假设。

在这里没有面试官来澄清问题，我们先来定义一些用例和约束条件。

### 用例

解决这个问题需要迭代的方法：1）**基准/负载测试**，2）**分析**瓶颈3）解决瓶颈，同时评估替代方案和取舍点。4）重复以上步骤，这是将一个基本的架构设计演变成可扩展架构设计的很好方式。

除非您有AWS的背景或正在申请需要AWS知识的职位，否则关于AWS的特定的细节信息并不需要了解。**本练习中讨论的许多原则可以更普遍地适用于AWS生态系统之外。**

#### 我们将问题的范围局限在以下用例

* **用户**进行读或写请求
	* **服务**处理，存储用户数据，然后返回结果
* **服务**需要从少量用户的服务发展到数百万的用户
	* 在架构演化到处理大量用户数据和请求的过程中，讨论通用的扩展模式
* **服务**具有高可用性

### 约束和假设

#### 陈述假设

* 流量分布不均匀
* 需要关系数据
* 从一个用户扩展到数千万用户
	* 将用户数量增加标记为：
		* 用户+
		* 用户++
		* 用户+++
		* ...
	* 1000万用户
	* 每月10亿写入操作
	* 每月1000亿读取操作
	* 读写比100: 1
	* 每次写入1KB内容

#### 计算用量

**假设不清楚时，向面试官澄清你是否需要估算数据。**

* 每月1TB新增内容
	* 每个写入1KB * 每月写入10亿次
	* 3年内有36TB的新增内容
	* 假设大多数写入来自新内容，而不是对现有内容的更新
* 平均每秒400次写入
* 平均每秒40000次读取

转换指南：

* 每月有250万秒
* 每秒1个请求=每月250万个请求
* 每秒40个请求=每月1亿个请求  
* 每秒400个请求=每月10亿个请求

## 步骤2：创建一个概要设计

>描述一个包含所有重要模块的概要设计。

![Imgur](http://i.imgur.com/B8LDKD7.png)

## 步骤3：设计核心模块

>详细介绍每个核心组件。

### 用例：用户进行读写请求

#### 目标

* 只有1-2个用户，您只需要一个基本的配置
	* 单服务器
	* 需要时垂直扩展
	* 监控以确定瓶颈

#### 从单机开始

* ** EC2上的Web服务器**
	* 存储用户数据
	* [** MySQL数据库**](https://github.com/donnemartin/system-design-primer#sql)

使用**垂直缩放**：

* 只需选择一个性能更强的服务器
* 密切关注性能指标，以确定如何扩大规模
	* 使用基本的监控来确定瓶颈：CPU，内存，IO，网络等
	* CloudWatch，top，nagios，statsd，graphite等
* 垂直扩展非常昂贵
* 无冗余/故障转移

*取舍，替代方案及其他细节：*

* **垂直扩展**的替代方案是[**水平扩展**](https://github.com/donnemartin/system-design-primer#horizontal-scaling)

#### 从SQL开始，考虑使用NoSQL

假设有关系型数据的需求。我们可以在单服务器中使用**MySQL数据库**。

*取舍，替代方案及其他细节：*
* 参见[关系数据库管理系统(RDBMS)](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms)部分
* 讨论使用[SQL或NoSQL](https://github.com/donnemartin/system-design-primer#sql-or-nosql)的原因

#### 分配公开静态IP

* Elastic IPs 提供一个公开接入点，其IP在重新启动时不会改变
* 帮助进行故障切换，只需将域名指向新的IP

#### 使用DNS

添加一个**DNS**，如Route 53，将域名映射到实例的公共IP。

*取舍，替代方案及其他细节：*

* 请参阅[域名系统](https://github.com/donnemartin/system-design-primer#domain-name-system)部分

#### 保护web服务器

* 只打开必需的端口
	* 允许Web服务器响应来自以下接口请求：
		* 80用于HTTP
		* 443用于HTTPS
		* 只允许白名单IP通过SSH登录22端口
	* 防止Web服务器访问站外链接

*取舍，替代方案及其他细节：*

* 请参阅[安全性](https://github.com/donnemartin/system-design-primer#security)部分

## 步骤4：扩展设计

>在约束条件下，找到并解决瓶颈。

### 用户+

![Imgur](http://i.imgur.com/rrfjMXB.png)

#### 假设

我们的用户数量开始上升，单个服务器上的负载持续增加。我们的**基准/负载测试**和**分析**表明**MySQL数据库**占用越来越多的内存和CPU资源，而用户内容正在占满磁盘空间。

目前为止，我们已经能够使用**垂直扩展**来解决这些问题。不幸的是，这个策略非常昂贵并且不能分别扩展**MySQL数据库**和**Web服务器**。

#### 目标

* 减轻单个服务器上的负载，可以分别扩展数据库和服务器
	* 将静态内容存储在**对象存储**
	* 将**MySQL数据库**移动到一个单独的服务器
* 缺点
	* 这些更改会增加复杂性，并且需要更改**Web服务器**以指向**对象存储**和**MySQL数据库**
	* 必须采取额外的安全措施来确保新模块的安全
	* AWS的成本也可能会增加，但应该与自己管理类似系统的成本进行权衡

#### 将静态内容分离

* 考虑使用像S3一样的托管**对象存储**来存储静态内容
	* 高度可扩展和可靠
	* 服务器端加密
* 将静态内容存储到S3
	* 用户文件
	* JS
	* CSS
	* 图片
	* 视频

#### 将MySQL数据库移动到单独的服务器中

* 考虑使用像RDS这样的服务来管理**MySQL数据库**
	* 简化管理，可扩展
	* 多个可用区域
	* 数据加密 [译注](https://en.wikipedia.org/wiki/Data_at_rest#Encryption)

#### 保护系统

* 在传输和持久化过程中加密数据
* 使用虚拟网络
	* 为单个**Web服务器**创建一个公共子网，以便它可以从互联网发送和接收数据
	* 创建一个私有子网，防止外部访问
	* 为每个模块只打开白名单IP的端口
* 在本练习的其余部分，应所有新加模块采取相同的模式

*取舍，替代方案及其他细节*

* 请参阅[安全性](https://github.com/donnemartin/system-design-primer#security)部分

### 用户++

![Imgur](http://i.imgur.com/raoFTXM.png)

#### 假设

我们的**基准/负载测试**和**分析**显示，的单个**Web服务器**在高峰时段出现瓶颈，导致响应缓慢，在某些情况下停机。随着服务的成熟，我们可以开始考虑高可用性和冗余方案。

#### 目标

* 以下目标尝试解决**Web服务器**的可扩展问题
	* 根据**基准/负载测试**和**分析**，您可能只需要实现这些技术中的一个或两个
* 使用[**水平扩展**](https://github.com/donnemartin/system-design-primer#horizontal-scaling)来处理增加的负载的能力并解决单点故障
	* 添加一个[**负载均衡**](https://github.com/donnemartin/system-design-primer#load-balancer)，如亚马逊的ELB或HAProxy
		* ELB是高可用的
		* 如果搭建自己的**负载均衡**，请在多个可用区配置成[双主](https://github.com/donnemartin/system-design-primer#active-active)或[主从](https://github.com/donnemartin/system-design-primer#active-passive)模式以提高可用性
		* 在**负载均衡**这一层面终止SSL，以减少后端服务器的计算负载，并简化证书管理
	* 使用多个**Web服务器**分布在多个可用区域
	* 在多个可用区域中，在[**主从动故障切换**](https://github.com/donnemartin/system-design-primer#master-slave-replication)模式下使用多个**MySQL**实例，以增加冗余
	* 将**Web服务器**从[**应用程序服务器**](https://github.com/donnemartin/system-design-primer#application-layer)中分离出来
		* 分别配置和扩展Web层和应用层
		* **Web服务器**可以作为[**反向代理**](https://github.com/donnemartin/system-design-primer#reverse-proxy-web-server)
		* 例如，您可以添加**应用程序服务器**专门处理**读取API**，而其他负责处理**写入API**
	* 将静态（和一些动态）内容移动到[**内容传送网络（CDN）**](https://github.com/donnemartin/system-design-primer#content-delivery-network)，来降低负载和延迟

*取舍，替代方案及其他细节：*

* 有关详细信息，请参阅上述链接内容

###用户+++

![Imgur](http://i.imgur.com/OZCxJr0.png)

**注意：** 为减少图的复杂程度，**内部负载均衡**没有画出

#### 假设

我们的**基准/负载测试**和**分析**显示我们的应用读取请求多于写入请求（读写比100：1），过多的读取请求严重降低了数据库的性能。

#### 目标

* 以下目标尝试解决**MySQL数据库**的可扩展问题
	* 基于**基准/负载测试**和**分析**，您可能只需要实现这些技术中的一个或两个
* 将以下数据移动到[**Memory Cache**](https://github.com/donnemartin/system-design-primer#cache)，如Elasticache，以减少负载和延迟：
	* 经常从**MySQL**读取的内容
		* 首先，在应用**Memory Cache**之前先尝试配置**MySQL数据库**缓存，来调查是否足以解决现存的瓶颈
	* 来自**Web服务器**的会话数据
		* **Web服务器**是无状态的，允许**自动扩展**
	* 从内存中读取1MB大约需要250微秒，而从SSD读取4倍的时间，而从磁盘读取需要80倍的时间。
* 添加[**MySQL读取副本**](https://github.com/donnemartin/system-design-primer#master-slave-replication)以减少写入服务器的负载
* 添加更多**Web服务器**和**应用程序服务器**以提高响应能力

*取舍，替代方案及其他细节：*

* 有关详细信息，请参阅上述链接内容

#### 添加MySQL读取副本

* 除了添加和扩展**Memory Cache**，**MySQL读副本**还可以帮助减轻对**MySQL master写入节点**的负载
* 修改**Web服务器**，区分写入请求和读取请求
* 在**MySQL读副本**添加**负载均衡**（未在图中标示）
* 大多数服务都是重读取，而不是重写入

*取舍，替代方案及其他细节：*

* 参见[关系数据库管理系统（RDBMS）](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms)部分

### 用户++++

![Imgur](http://i.imgur.com/3X8nmdL.png)

####假设

我们的**基准/负载测试**和**分析**显示，我们的流量在美国的工作时间达到高峰，当用户离开办公室时流量显着下降，可以通过根据实际负载自动启动和关闭服务器来降低成本。我们是一个小商店，所以我们希望尽可能自动化地完成**自动缩放**和一般DevOps运维操作。

####目标

* 根据需要添加**自动扩展**功能以提高系统容量
	* 在流量顶峰正常工作
	* 通过关闭未使用的实例来降低成本
* 自动化DevOps
	* Chef, Puppet, Ansible等
* 继续监控指标来找出系统瓶颈
	* **主机级别** - 评估单个EC2实例的表现
	* **综合等级** - 查看负载均衡的统计信息
	* **日志分析** - CloudWatch，CloudTrail，Loggly，Splunk，Sumo
	* **外部网站分析工具** - Pingdom或New Relic
	* **处理通知和事件** - PagerDuty
	* **错误报告** - Sentry

#### 添加自动缩放

* 考虑使用一个托管服务，如AWS **Autoscaling**
	* 为每个**Web服务器**创建一个组，并为每个**应用程序服务器**类型创建一个组，并将每个组放置在多个可用区域中
	* 设置最小和最大数量的实例
	* 通过CloudWatch触发启动和关闭实例
		* 根据一天内不同时间段的历史负载数据，或
		* 一段时间内的指标：
			* CPU负载
			* 网络延迟
			* 网络流量
			* 自定义指标
	* 缺点
		* 自动缩放可以引入复杂性
		* 系统可能需要一段时间才能扩大容量来处理增加的请求，或者在请求降低的时候缩减容量

### 用户+++++

![Imgur](http://i.imgur.com/jj3A5N8.png)

**注意：** **自动缩放**组未标示

#### 假设

随着服务的不断扩大，我们继续迭代运行**基准/负载测试**和**剖析**来发现和解决新的瓶颈。

#### 目标

鉴于问题的约束，我们将继续解决可扩展问题：

* 如果我们的**MySQL数据库**过于庞大，我们可能只考虑在数据库中存储有限时间段内数据，而将其余数据存储在数据仓库中，如Redshift
	* Redshift等数据仓库可以轻松地处理每月1TB新增内容
* 如果每秒有4万次读取请求，通过扩展**Memory Cache**集群可以很好的应对经常被访问内容的读取流量，这样做对于处理不均匀分布的流量和流量峰值也很有用
	* **SQL读取副本**可能在处理缓存未命中时遇到问题，我们需要使用其他SQL扩展模式
* 平均每秒400次写入请求（在流量峰值时可能更高），对于单个**SQL Master-Slave**写入节点来说可能是困难的，同样地也表明需要额外的扩展技术

SQL扩展模式包括：

* [联合](https://github.com/donnemartin/system-design-primer#federation)
* [分片](https://github.com/donnemartin/system-design-primer#sharding)
* [反规范化](https://github.com/donnemartin/system-design-primer#denormalization)
* [SQL调优](https://github.com/donnemartin/system-design-primer#sql-tuning)

为了进一步解决高读写请求，我们还应考虑将适当的数据移动到[**NoSQL数据库**](https://github.com/donnemartin/system-design-primer#nosql)，如DynamoDB。

我们可以进一步分离我们的[**应用服务器**](https://github.com/donnemartin/system-design-primer#application-layer)，以允许独立扩展。可以把不需要实时完成计算任务通过**队列**和**Worker**用批处理或[**异步**](https://github.com/donnemartin/system-design-primer#asynchronism)的方式来完成：

* 例如，在照片服务中，照片上传和缩略图创建可以分开：
	* **客户**上传照片
	* **应用服务器**将任务放在**Queue**（如SQS）中
	* **任务服务**在EC2或Lambda读取工作的**队列**然后：
		* 创建缩略图
		* 更新**数据库**
		* 将缩略图存储在**Object Store**

*取舍，替代方案及其他细节：*

* 有关详细信息，请参阅上述链接内容

##其他要点

>根据问题范围和剩余时间，深入其他主题。

### SQL缩放模式

* [读取副本](https://github.com/donnemartin/system-design-primer#master-slave)
* [联合](https://github.com/donnemartin/system-design-primer#federation)
* [分片](https://github.com/donnemartin/system-design-primer#sharding)
* [反规范化](https://github.com/donnemartin/system-design-primer#denormalization)
* [SQL调优](https://github.com/donnemartin/system-design-primer#sql-tuning)

#### NoSQL

* [键值存储](https://github.com/donnemartin/system-design-primer#key-value-store)
* [文件存储](https://github.com/donnemartin/system-design-primer#document-store)
* [表格存储](https://github.com/donnemartin/system-design-primer#wide-column-store)
* [图数据库](https://github.com/donnemartin/system-design-primer#graph-database)
* [SQL vs NoSQL](https://github.com/donnemartin/system-design-primer#sql-or-nosql)

###缓存

* 在哪里缓存
	* [客户端缓存](https://github.com/donnemartin/system-design-primer#client-caching)
	* [CDN缓存](https://github.com/donnemartin/system-design-primer#cdn-caching)
	* [Web服务器缓存](https://github.com/donnemartin/system-design-primer#web-server-caching)
	* [数据库缓存](https://github.com/donnemartin/system-design-primer#database-caching)
	* [应用缓存](https://github.com/donnemartin/system-design-primer#application-caching)

* 缓存什么

	* [在数据库查询级别缓存](https://github.com/donnemartin/system-design-primer#caching-at-the-database-query-level)
	* [在对象级缓存](https://github.com/donnemartin/system-design-primer#caching-at-the-object-level)

* 何时更新缓存
	* [Cache-aside](https://github.com/donnemartin/system-design-primer#cache-aside)
	* [Write-through](https://github.com/donnemartin/system-design-primer#write-through)
	* [Write-back（write-back）](https://github.com/donnemartin/system-design-primer#write-behind-write-back)
	* [Refresh ahead](https://github.com/donnemartin/system-design-primer#refresh-ahead)

###异步和微服务

* [消息队列](https://github.com/donnemartin/system-design-primer#message-queues)
* [任务队列](https://github.com/donnemartin/system-design-primer#task-queues)
* [限流](https://github.com/donnemartin/system-design-primer#back-pressure)
* [微服务](https://github.com/donnemartin/system-design-primer#microservices)

### 通讯

讨论取舍：

* 与外部通讯 - [REST风格的HTTP API](https://github.com/donnemartin/system-design-primer#representational-state-transfer-rest)
* 内部通讯 - [RPC](https://github.com/donnemartin/system-design-primer#remote-procedure-call-rpc)
* [服务发现](https://github.com/donnemartin/system-design-primer#service-discovery)

### 安全

* 请参阅[安全部分](https://github.com/donnemartin/system-design-primer#security)。

### 延迟数据

请参阅[每个程序员应该知道的延迟数据](https://github.com/donnemartin/system-design-primer#latency-numbers-every-programmer-should-know)。

### 正在进行

* 继续对系统进行基准测试和监控，以解决瓶颈问题
* 扩展是一个迭代过程

