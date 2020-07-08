---
layout: post
title: 【spring boot 系列】SpringBoot 集成 RocketMQ
categories: Zookeeper Spring
description: SpringBoot 集成 RocketMQ
keywords: RocketMQ, 中间件, Apache, Spring, Spring boot
---

RcoketMQ 是一款低延迟、高可靠、可伸缩、易于使用的消息中间件。

具有以下特性：
- 支持发布/订阅（Pub/Sub）和点对点（P2P）消息模型
- 能够保证严格的消息顺序，在一个队列中可靠的先进先出（FIFO）和严格的顺序传递
- 提供丰富的消息拉取模式，支持拉（pull）和推（push）两种消息模式
- 单一队列百万消息的堆积能力，亿级消息堆积能力
- 支持多种消息协议，如 JMS、MQTT 等
- 分布式高可用的部署架构,满足至少一次消息传递语义


