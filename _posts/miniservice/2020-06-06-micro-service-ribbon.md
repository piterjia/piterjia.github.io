---
layout: post
title: 【Spring Cloud系列】Ribbon 工作原理与负载均衡策略
categories: 微服务
description: Ribbon 工作原理与负载均衡策略
keywords: Ribbon, spring Cloud, distributed, micro service
---


目前主流的负载方案分为两种，一种是集中式负载均衡，在消费者和服务提供方中间使用独立的代理方式进行负载，有硬件的，比如F5，也有软件的，比如Nginx。
另一种则是客户端自己做负载均衡，根据自己的请求情况做负载，Ribbon 就是属于客户端自己做负载的。

Ribbon 是 Netflix 开源的一款用于客户端负载均衡的工具软件。默认的策略是轮询，我们可以自定义负载策略来覆盖默认的，当然也可以通过配置指定使用哪些策略。

本文主要讲解，Ribbon的特点以及工作模型，以及负载均衡策略。

Ribbon 的特点：
- 丰富的组件库：整套负载均衡由7个具体策略组成，不管你是什么特殊需求，都有合适的策略供你选择
- 适配性好：SpringCloud里的五小强（eureka，feign，gateway，zuul，hystrix），谁拿都能用


## Ribbon 工作模型

![](/images/microservice/ribbon-introduce-1.png)

如图所示：

一个 HttpRequest 请求发过来，先被转发到 Eureka 上。此时 Eureka 仍然通过服务发现获取了所有服务节点的物理地址，他不知道该调用哪一个服务节点比较好，只好把请求转到了 Ribbon 上。

然后 Ribbon 主要通过以下两种模式进行工作：
- **IPing**  IPing 是 Ribbon 的一套 healthcheck 机制。故名思议，就是要 Ping 一下目标机器看是否还在线，一般情况下 IPing 并不会主动向服务节点发起 healthcheck 请求，Ribbon 后台通过静默处理返回true默认表示所有服务节点都处于存活状态（和Eureka集成的时候会检查服节点UP状态）。
- **IRule**  Ribbon的组件库，各种负载均衡策略都继承自IRule接口。所有经过 Ribbon 的请求都会先请示IRule 一把，找到负载均衡策略选定的目标机器，然后再把请求转发出去。


## 负载均衡策略

负载均衡在系统架构中是一个非常重要，并且是不得不去实施的内容。因为负载均衡是对系统的高可用、网络压力的缓解和处理能力扩容的重要手段之一。我们通常所说的负载均衡都指的是服务端负载均衡。


| 策略类     | 命名 | 描述 |
|:---------|:---------------|:---------------|
| RandomRule  | 随机策略            | 随机选择server        |
| RoundRobinRule | 轮询策略          | 按照顺序选择server（ribbon默认策略）         |
| RetryRule   | 重试策略            | 在一个配置时间段内，当选择server不成功，则一直尝试选择一个可用的server            |
| BestAvailableRule   | 最低并发策略     | 逐个考察server，如果server断路器打开，则忽略，再选择其中并发链接最低的server  |
| AvailabilityFilteringRule   | 可用过滤策略	| 过滤掉一直失败并被标记为circuit tripped的server，过滤掉那些高并发链接的server（active connections超过配置的阈值）            |
| ResponseTimeWeightedRule   | 响应时间加权重策略	            | 根据server的响应时间分配权重，响应时间越长，权重越低，被选择到的概率也就越低。响应时间越短，权重越高，被选中的概率越高，这个策略很贴切，综合了各种因素，比如：网络，磁盘，io等，都直接影响响应时间|
| ZoneAvoidanceRule   | 区域权重策略	            | 综合判断server所在区域的性能，和server的可用性，轮询选择server并且判断一个AWS Zone的运行性能是否可用，剔除不可用的Zone中的所有server|


### 微服务架构中使用客户端负载均衡
通过 Spring Cloud Ribbon 的封装，我们在微服务架构中使用客户端负载均衡，只需要如下两步：
- 服务提供者，只需要启动多个服务实例并注册到一个注册中心或是多个相关联的服务注册中心。
- 服务消费者，直接通过调用被 @LoadBalanced 注解修饰过的 RestTemplate 来实现面向服务的接口调用。
  
这样，我们就可以将服务提供者的高可用以及服务消费者的负载均衡调用一起实现了。