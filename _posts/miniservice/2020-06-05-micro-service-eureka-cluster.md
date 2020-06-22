---
layout: post
title: 【Spring Cloud系列】搭建 Eureka 高可用服务注册中心集群
categories: 微服务
description: 搭建 Eureka 高可用服务注册中心集群
keywords: Eureka, spring Cloud, distributed, micro service
---

Eureka 是 spring cloud 中的一个负责服务注册与发现的组件。
在进入本文之前，强烈建议先去熟悉一下上一篇文章，单节点的 Eureka 介绍。

[单节点 Eureka](https://piterjia.github.io/2020/06/04/micro-service-eureka/)

[单节点 Eureka 源码](https://github.com/piterjia/eurekademo/tree/master/stage1)


一个 Eureka 中分为 eureka server 和 eureka client。其中 eureka server 是作为服务的注册与发现中心。 eureka client 既可以作为服务的生产者，又可以作为服务的消费者。

该案例，我们采用一个项目中集成多个子 module 的方式实现。

1. 第一个Module:  Eureka server，该 eureka server 同步保留着所有的服务信息。该子module命名为 eurekaserver 。
2. 第二个Module:  Eureka server，该 eureka server 同步保留着所有的服务信息。该子module命名为 eurekaserver02 。
3. 第三个Module:  Eureka server，该 eureka server 同步保留着所有的服务信息。该子module命名为 eurekaserver03 。
4. 第四个Module: eureka client，向服务端发起服务注册，他作为服务提供者。该子module命名为 eurekaprovider 。
5. 第五个Module: eureka client，向服务端发起服务注册，他作为服务消费者。该子module命名为 eurekaconsumer 。

其中：eurekaserver，eurekaserver02,eurekaserver03 互相备份，当服务提供者向其中一个 Eureka 注册服务时，这个服务就会被共享到其他 Eureka上，这样所有的 Eureka 都会有相同的服务。

## 科普
可能有很多人对分布式和集群这两个概念有点混淆。我先用通俗易懂的话给大家解释下：
- 分布式：一个业务分拆多个子业务，部署在不同的服务器上
- 集群：同一个业务，分别部署在不同的服务器上

所以分布式的每一个节点，完成的是不同的业务，一个节点挂了，那么这个业务功能就无法访问了，甚至可能会影响到其他业务。而集群是一个比较有组织的架构，正因为有组织性，一个服务节点挂了，其他服务节点可以顶上来，从而保证了服务的健壮性。

## 父 Module 项目结构
spring cloud 基于 spring boot 进行开发，因此我们需要创建一个 spring boot 项目，同时在里面新建5个子 module，分别是：eurekaserver、eurekaserver02、eurekaserver03、eurekaprovider、eurekaconsumer。

项目结构如下：

![](/images/microservice/eureka-cluster-1.png)



## 搭建 Eureka 集群

在搭建 Eureka 集群之前，先来回顾一下前面搭建的单个 Eureka 服务，[单节点 Eureka](https://piterjia.github.io/2020/06/04/micro-service-eureka/)。看下 yml 配置文件：
```
spring:
  application:
    #服务名称
    name: eureka-server
server:
  #服务注册中心端口号
  port: 10000
eureka:
  instance:
    hostname: eureka01
  client:
    #是否向服务注册中心注册自己
    registerWithEureka: false
    #是否检索服务
    fetchRegistry: false
    #服务注册中心的配置内容，指定服务注册中心位置
    #注册中心路径，如果有多个eureka server，在这里需要配置其他eureka server的地址，用","进行区分，如"http://address:8888/eureka,http://address:8887/eureka"
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  # 以下配置信息可选
  server:
    #开启注册中心的保护机制，默认是开启
    enable-self-preservation: true
    #设置保护机制的阈值，默认是0.85。
    renewal-percent-threshold: 0.8
```

这是一个 Eureka 服务，名称是 eureka01 注册中心是它自己。

那么我们如何去搭建一个 Eureka 集群呢？假设现在有三个 Eureka 服务： eureka01 、 eureka02 和  eureka03。

为了体现出集群的你中有我，我中有你，不难想象， eureka01 中应该挂上 eureka02 和 eureka03 ； eureka02 中应该挂上 eureka01 和 eureka03 ； eureka03 中应该挂上 eureka01 和 eureka02。它们三者之间互为备份。如下图所示：

![](/images/microservice/eureka-cluster-5.png)

那么代码层面，如何实现呢？


### Eureka01 的改造

由上面的分析可知，Eureka01 需要挂上 Eureka02 和 Eureka03，所以在 Eureka01 的配置文件中需要重新配置一下 defaultZone，如下：
```
spring:
  application:
    #服务名称
    name: eureka-server-01
server:
  #服务注册中心端口号
  port: 20000
eureka:
  instance:
    hostname: eureka01
  client:
    #是否向服务注册中心注册自己
    registerWithEureka: false
    #是否检索服务
    fetchRegistry: false
    #服务注册中心的配置内容，指定服务注册中心位置
    #注册中心路径，如果有多个eureka server，在这里需要配置其他eureka server的地址，用","进行区分，如"http://address:8888/eureka,http://address:8887/eureka"
    serviceUrl:
      defaultZone: http://eureka01:20000/eureka/,http://eureka02:20001/eureka/,http://eureka03:20002/eureka/
  server:
    #开启注册中心的保护机制，默认是开启
    enable-self-preservation: true
    #设置保护机制的阈值，默认是0.85。
    renewal-percent-threshold: 0.8
```

主要的改动是：
```
serviceUrl:
    defaultZone: http://eureka01:20000/eureka/,http://eureka02:20001/eureka/,http://eureka03:20002/eureka/
```

OK，defaultZone 配置好了 eureka02 和 eureka03。


### 搭建 Eureka02

基于 eureka01 这个子Module ，复制一个子 module，命名为 eurekaserver02。

修改其配置文件：
```
spring:
  application:
    #服务名称
    name: eureka-server-02
server:
  #服务注册中心端口号
  port: 20001
eureka:
  instance:
    hostname: eureka02
  client:
    #是否向服务注册中心注册自己
    registerWithEureka: false
    #是否检索服务
    fetchRegistry: false
    #服务注册中心的配置内容，指定服务注册中心位置
    #注册中心路径，如果有多个eureka server，在这里需要配置其他eureka server的地址，用","进行区分，如"http://address:8888/eureka,http://address:8887/eureka"
    serviceUrl:
      defaultZone: http://eureka01:20000/eureka/,http://eureka02:20001/eureka/,http://eureka03:20002/eureka/
      # defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  server:
    #开启注册中心的保护机制，默认是开启
    enable-self-preservation: true
    #设置保护机制的阈值，默认是0.85。
    renewal-percent-threshold: 0.8
```

在 eureka02 中，把 eureka01 和 eureka03 挂进来。


### 搭建 Eureka03

同样的，基于 eureka01 子Module，复制一个module，命名为 eurekaserver03。

修改其配置文件：
```
spring:
  application:
    #服务名称
    name: eureka-server-03
server:
  #服务注册中心端口号
  port: 20002
eureka:
  instance:
    hostname: eureka03
  client:
    #是否向服务注册中心注册自己
    registerWithEureka: false
    #是否检索服务
    fetchRegistry: false
    #服务注册中心的配置内容，指定服务注册中心位置
    #注册中心路径，如果有多个eureka server，在这里需要配置其他eureka server的地址，用","进行区分，如"http://address:8888/eureka,http://address:8887/eureka"
    serviceUrl:
      defaultZone: http://eureka01:20000/eureka/,http://eureka02:20001/eureka/,http://eureka03:20002/eureka/
      # defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  server:
    #开启注册中心的保护机制，默认是开启
    enable-self-preservation: true
    #设置保护机制的阈值，默认是0.85。
    renewal-percent-threshold: 0.8
```

现在三个 eureka 注册中心都搭建好了，最后别忘了在本地 hosts 文件中将 eureka01、eureka02 和 eureka03 映射到 127.0.0.1。

分别启动这三个服务，可以发现，这个三个 eureka 互为备份。

![](/images/microservice/eureka-cluster-2.png)


## eureka provider 修改

这个 provider 前一篇文章  [单节点 Eureka](https://piterjia.github.io/2020/06/04/micro-service-eureka/) 已经详细介绍过他的职责和作用。简单概述：该 eureka client 的职责是：向服务端发起服务注册，他作为服务提供者对外提供服务。

### 修改配置信息
之前的配资配置信息如下：
```
#服务端口
server.port=10001
#服务名称
spring.application.name=eureka-provider
#服务地址
eureka.instance.hostname=localhost
#注册中心路径，表示我们向这个注册中心注册服务，如果向多个注册中心注册，用“，”进行分隔
eureka.client.service-url.defaultZone=http://eureka01:20000/eureka/
#心跳间隔10s，默认30s。每一个服务配置后，心跳间隔和心跳超时时间会被保存在server端，不同服务的心跳频率可能不同，server端会根据保存的配置来分别探活
eureka.instance.lease-renewal-interval-in-seconds=10
#心跳超时时间30s，默认90s。从client端最后一次发出心跳后，达到这个时间没有再次发出心跳，表示服务不可用，将它的实例从注册中心移除
eureka.instance.lease-expiration-duration-in-seconds=30
```

之前是将该服务注册到 eureka01 中，因为之前就这一个 eureka 注册中心。那么现在有三个注册中心，我们需要修改下配置，将该服务注册到三个 eureka 中。

defaultZone 更改如下，其他的配置信息不变：
```
#注册中心路径，表示我们向这个注册中心注册服务，如果向多个注册中心注册，用“，”进行分隔
eureka.client.service-url.defaultZone=http://eureka01:20000/eureka/,http://eureka02:20001/eureka/,http://eureka03:20002/eureka/

```

启动 provider服务，并成功挂载到这个三个 eureka 组成的集群中：

![](/images/microservice/eureka-cluster-3.png)



## eureka consumer 修改

这一个 eureka client 的职责是：向服务端发起服务注册，他作为服务消费者者，对 provider 进行远程调用。详见  [单节点 Eureka](https://piterjia.github.io/2020/06/04/micro-service-eureka/) 

### 修改配置信息
之前的配资配置信息如下：
```
#服务端口
server.port=10002
#注册到Eureka Server上的服务ID
spring.application.name=eureka-client
#服务地址
eureka.instance.hostname=localhost
#Eureka Server默认地址：注册中心路径，表示我们向这个注册中心注册服务，如果向多个注册中心注册，用“，”进行分隔
eureka.client.service-url.defaultZone=http://eureka01:20000/eureka
#心跳间隔10s，默认30s。每一个服务配置后，心跳间隔和心跳超时时间会被保存在server端，不同服务的心跳频率可能不同，server端会根据保存的配置来分别探活
eureka.instance.lease-renewal-interval-in-seconds=10
#心跳超时时间30s，默认90s。从client端最后一次发出心跳后，达到这个时间没有再次发出心跳，表示服务不可用，将它的实例从注册中心移除
eureka.instance.lease-expiration-duration-in-seconds=30
```


之前是将该服务注册到 eureka01 中，因为之前就这一个 eureka 注册中心。那么现在有三个注册中心，我们需要修改下配置，将该服务注册到三个 eureka 中。

defaultZone 更改如下，其他的配置信息不变：
```
#注册中心路径，表示我们向这个注册中心注册服务，如果向多个注册中心注册，用“，”进行分隔
eureka.client.service-url.defaultZone=http://eureka01:20000/eureka/,http://eureka02:20001/eureka/,http://eureka03:20002/eureka/

```

启动 consumer，并成功挂载到这个三个 eureka 组成的集群中：

![](/images/microservice/eureka-cluster-4.png)



### consumer远程调用
至此，所有的准备工作已经做完了。浏览器直接调用，或者使用postman等调试工具，发起远程调用：
```
http://localhost:10002/hi
```

成功获取到了 provider 提供的服务返回信息：

![](/images/microservice/eureka-demo-7.png)


## 总结：

本文简单介绍了 eureka 的集群模式，使得 eureka 服务注册中心实现了高可用。

该案例源码地址：

[集群 Eureka 源码](https://github.com/piterjia/eurekademo/tree/master/stage2)
