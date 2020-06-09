---
layout: post
title: RocketMQ 简介以及集群模式
categories: Distributed
description: RocketMQ 简介以及集群模式
keywords: RocketMQ, distributed 
---
以 RocketMQ 的事务消息功能为示例，总结分布式事务的内容。并介绍如何搭建一套生产级的RocketMQ消息集群。

# RocketMQ 介绍

RocketMQ 是阿里开源的并贡献给 Apache 基金会的一款分布式消息平台，它具有低延迟、高性能和可靠性、万亿级容量和灵活的可伸缩性的特点，单机也可以支持亿级的消息堆积能力、单机写入 TPS 单实例约7万条/秒，单机部署3个 Broker，可以跑到最高12万条/秒。

基于以上强大的性能，以及阿里的技术影响力，RocketMQ 目前在国内互联网公司中被使用得比较广泛。

整个 RocketMQ 消息系统主要由如下4个部分组成：

**从中间件服务角度来看整个RocketMQ消息系统（服务端）主要分为：NameSrv和Broker两个部分。**

## NameSrv：
在 RocketMQ 分布式消息系统中，NameSrv 主要提供两个功能：
- 提供服务发现和注册：主要是管理 Broker，NameSrv 接受来自 Broker 的注册，并通过心跳机制来检测 Broker 服务的健康。
- 提供路由功能：集群(这里是指以集群方式部署的 NameSrv )中的每个 NameSrv 都保存了 Broker 集群(这里是指以集群方式部署的 Broker )中整个的路由信息和队列信息。注意，在 NameSrv 集群中，每个 NameSrv 都是相互独立的，所以每个 Broker 需要连接所有的 NameSrv，每创建一个新的 topic 都要同步到所有的 NameSrv 上。

## Broker：
主要是负责消息的存储、传递、查询以及高可用(HA)保证等。其由如下几个子模块构成：
- remoting，是 Broker 的服务入口，负责客户端的接入（Producer和Consumer）和请求处理。
- client，管理客户端和维护消费者对于 Topic 的订阅。
- store，提供针对存储和消息查询的简单的 API （数据存储在物理磁盘）。
- HA， 提供数据在主从节点间同步的功能特性。
- Index，通过特定的 key 构建消息索引，并提供快速的索引查询服务。


**而从客户端的角度看主要有：Producer、Consumer两个部分。**

## Producer：消息生产者
消息的生产者，由用户进行分布式部署，消息由 Producer 通过多种负载均衡模式发送到 Broker 集群，发送低延时，支持快速失败。

## Consumer：消息消费者
消息的消费者，也由用户部署，支持 PUSH 和 PULL 两种消费模式，支持集群消费和广播消息，提供实时的消息订阅机制，满足大多数消费场景。

## 总结
整个 RocketMQ 消息集群就是由 NameSrv/Broker、Producer/Consumer 组成的。以一条完整的信息流转为例，来看看 RocketMQ 消息系统是如何运转的，如下图所示：
![](/images/posts/distributed/rocketmq1.png)


下面我们就具体看看，如何搭建一套生产级的 RocketMQ 消息集群。

# RocketMQ集群模式

对于 NameSrv 来说可以同时部署多个节点，并且这些节点间也不需要有任何的信息同步，这里因为每个NameSrv节点都会存储全量路由信息，在NameSrv集群模式下，每个 Broker 都需要同时向集群中的每个 NameSrv 节点发送注册信息，所以这里对于 NameSrv 的集群部署来说并不需要做什么额外的设置。

## Broker 集群模式
而对于 Broker 集群来说就有多种模式了，先介绍下这几种模式，然后再看生产级的部署方式.

### 1）、单个Master模式   
一个Broker作为主服务，不设置任何Slave，很显然这种方式风险比较大，存在单节点故障会导致整个基于消息的服务挂掉，所以生产环境不可能采用这种模式。

###  2）、多Master模式  
这种模式的 Broker 集群，全是 Master，没有 Slave 节点。这种方式的优缺点如下：

优点：配置会比较简单一些，如果单个 Master 挂掉或重启维护的话对应用是没有什么影响的。

缺点：就是在单台机器宕机期间，这台机器上未被消费的消息在机器恢复之前是不可以进行消息订阅的，这对消息的实时性会有一些影响。

### 3）、多Master多Slave模式（异步复制）

在这种模式下 Broker 集群存在多个 Master 节点，并且每个 Master 节点都会对应一个 Slave 节点，有多对 Master-Slave，HA(高可用)之间采用异步复制的方式进行信息同步，在这种方式下主从之间会有短暂的毫秒级的消息延迟。

优点：在这种模式下即便磁盘损坏了，消息丢失的情况也非常少，因为主从之间有信息备份；并且，在这种模式下消息的实时性也不会受影响，因为 Master 宕机后Slave 可以自动切换为 Master 模式，这样 Consumer 仍然可以通过 Slave 进行消息消费，而这个过程对应用来说则是完全透明的，并不需要人工干预；另外，这种模式的性能与多 Master 模式几乎差不多。

缺点：如果Master宕机，并且在磁盘损坏的情况下，会丢失少量的消息。

### 4）、多Master多Slave模式（同步复制）

这种模式与3）差不多，只是HA采用的是同步双写的方式，即主备都写成功后，才会向应用返回成功。

优点：在这种模式下数据与服务都不存在单点的情况，在Master宕机的情况下，消息也没有延迟，服务的可用性以及数据的可用性都非常高。

缺点：性能相比于异步复制来说略低一些（大约10%）；另外一个缺点就是相比于异步复制，目前 Slave 备机还暂时不能实现自动切换为 Master。


## 总结

综合考虑以上集群模式的优缺点，在实际生产环境中，目前基于RocketMQ消息集群的部署方式基本都是采用多Master多Slave（异步复制）这种模式。
