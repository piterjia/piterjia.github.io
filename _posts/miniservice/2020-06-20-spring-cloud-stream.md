---
layout: post
title: 【Spring Cloud 系列】 Stream 体系架构
categories: 微服务
description: Stream 体系架构
keywords: Spring Cloud, Stream, 体系架构, 交互模型
---


Stream 就是 Spring Cloud 中专门负责集成消息中间件的组件

## 什么是 Stream

Spring Cloud Stream 是基于 Spring Boot 构建的，专为构建消息驱动服务所设计的应用框架，它的底层使用 Spring Integration 来为消息代理层提供网络连接支持。

## Stream 的几个关键词

![](/images/microservice/stream1.png)

- 应用模型    Stream 提供了应用模型的抽象，引入了三个角色，分别是输入通道（Input）、输出通道（Output）和通道与底层中间件之间的代理（Binder）
- 适配层抽象   Stream 将组件与底层中间件之间的通信过程抽象成了 Binder 层，使得应用层不需要关心底层中间件是 Kafka 还是 RabbitMQ，只需要关注自身的业务逻辑就好
- 插件式适配层   Binder 层采用一种插件形式来提供服务（这和 Spring 的一贯设计思想一致），开发人员可以很方便的替换适配层或者开发自定义的适配逻辑
- 持久化的发布订阅模型   发布订阅是所有消息组件最核心的功能
- 消费组   Stream 允许将多个 Consumer 加入到一个消费组，如下图所示，4 个消费者分别被添加到 2 个不同的消费组中。消费组的作用是确保一条消息只能被组内的一台实例消费。如下图所示，一个新消息将被两个Group各消费一次


![](/images/microservice/stream2.png)


- 分区   Stream 支持在多个消费者实例之间创建分区，这样我们通过某些特征量做消息分发，保证相同标识的消息总是能被同一个消费者处理，如下图所示，Partition 1 和 Partition 2 的消息只会被指定的 Consumer 定向消费

![](/images/microservice/stream3.png)


## Stream体系架构


放上一张Stream官方文档的图，Stream的体系架构主要包括 Input、Output 和 Binder 三个部分，让我们分别了解一下图中的几个组件：

![](/images/microservice/stream4.png)

### Input通道

也就是输入通道，它的作用是将消息组件中获取到的Message传递给消费者进行消费。在Stream里我们可以借助 @Input 注解轻松定义一个输入通道：

```
public interface MyTopic {
    @Input
    SubscribableChannel input();
}
```

### Output通道

Output是输出通道，用来将生产者创建的新消息发送到对应的Topic中去，在Stream中我们可以借助 @Output 标签定义一个输出通道，@Output 和 @Input 可以放在一个接口中声明

```
public interface MyTopic {

	// 这里可以给Output自定义目标通道名称，比如@Output("myTarget")
    @Output
    MessageChannel output();

    @Input
    SubscribableChannel input();
}
```


### Binder

Stream提供了一个Binder抽象层，作为连接外部消息中间件（指RabbitMQ，Kafka）的桥梁，它的作用方式如下图所示。

![](/images/microservice/stream5.png)

- 中间那块浅绿色的大方块 Broker 就是消息中间件，围绕它周围的三个深绿色方块就是 Stream 中的 Binder 组件，针对每一个不同的 Broker（比如Kafka和RabbitMQ），Stream 都有一个对应的 Binder 具体实现来做适配。
- Binder 它作为一个适配层，对上层应用程序和底层消息组件之间做了一层屏障，我们的应用程序不用关注底层究竟是使用了 Kafka 还是 RabbitMQ，只管用注解开启响应的消息通道，剩下的事通通交给 Binder 来搞定。同样，当我们需要替换底层中间件的时候，只要变更 Binder 的依赖项，然后修改一些配置文件就好了，对我们的应用程序几乎是无感知的。


## 目的地绑定
通常来说 @Input 和@ Output 如果部署在同一个项目中的话，是不能起一样的名字的，否则 Spring 在启动阶段就会报错了。比如我们定义了 @Input(“myTopic”)，就不可能再定义一个同样名字的 @Output 注解。考虑到发布/监听的队列名称默认就是注解里所指定的名字，如果使用了不同的名字，那自然就不会在同一个消息队列中遇到，那么我们如何将作用于同一个Topic的生产者和消费者定义在一个项目中呢？

需要借助Stream的目的地绑定功能了，看下面一段配置：
```
spring.cloud.stream.bindings.<通道名>.destination=<主题名>
```

通过上面这段配置，我们可以将不同的通道绑定到指定的目的地，把这里的<通道名>替换成配置在 @Input 或@ Output 中的 name，然后再把<主题名>改成同一个 Topic，这样一来 Stream 就将对应的通道绑定到指定的 Topic 队列上。

