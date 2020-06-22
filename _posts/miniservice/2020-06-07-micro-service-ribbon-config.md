---
layout: post
title: 【Spring Cloud系列】Ribbon 常用配置
categories: 微服务
description: Ribbon 常用配置
keywords: Ribbon, spring Cloud, distributed, micro service
---

Ribbon 是 SpringCloud 下的客户端负载均衡组件，本文我们主要介绍 Ribbon 的一些常用配置和配置 Ribbon 的两种方式。


## 常用配置
### 1. 禁用 Eureka
当我们在 RestTemplate 上添加 @LoadBalanced 注解后，就可以用**服务名称**来调用接口了，当有多个服务的时候，还能做负载均衡。

这是因为 Eureka 中的服务信息已经被拉取到了客户端本地，如果我们不想和 Eureka 集成，可以通过下面的配置方法将其禁用。
```
# 禁用 Eureka
ribbon.eureka.enabled=false
```

当我们禁用了 Eureka 之后，就不能使用服务名称去调用接口了，必须指定服务地址。

### 2. 配置接口地址列表
上面我们讲了可以禁用 Eureka，禁用之后就需要手动配置调用的服务地址了，配置如下：
```
# 禁用 Eureka 后手动配置服务地址
ribbon-config-demo.ribbon.listOfServers=localhost:8081,localhost:8083
```

这个配置是针对具体服务的，前缀就是服务名称，配置完之后就可以和之前一样使用服务名称来调用接口了。


### 3. 配置负载均衡策略
Ribbon 默认的策略是轮询，从我们前面讲解的例子输出的结果就可以看出来，Ribbon 中提供了很多的策略，我们通过配置可以指定服务使用哪种策略来进行负载操作。

#### 代码配置 Ribbon
配置 Ribbon 最简单的方式就是通过配置文件实现。当然我们也可以通过代码的方式来配置。

通过代码方式来配置负载策略，首先需要创建一个配置类，初始化自定义的策略，代码如下所示。
```
@Configuration
public class BeanConfiguration {
    @Bean
    public MyRule rule() {
        return new MyRule();
    }
}
```
创建一个 Ribbon 客户端的配置类，关联 BeanConfiguration，用 name 来指定调用的服务名称，代码如下所示。
```
@RibbonClient(name = "ribbon-config-demo", configuration = BeanConfiguration.class)
public class RibbonClientConfig {
}
```
然后启动服务，访问接口即可看到和之前一样的效果。


#### 配置文件方式配置 Ribbon
除了使用代码进行 Ribbon 的配置，我们还可以通过配置文件的方式来为 Ribbon 指定对应的配置：
```
<clientName>.ribbon.NFLoadBalancerClassName: Should implement ILoadBalancer(负载均衡器操作接口)
<clientName>.ribbon.NFLoadBalancerRuleClassName: Should implement IRule(负载均衡算法)
<clientName>.ribbon.NFLoadBalancerPingClassName: Should implement IPing(服务可用性检查)
<clientName>.ribbon.NIWSServerListClassName: Should implement ServerList(服务列表获取)
<clientName>.ribbon.NIWSServerListFilterClassName: Should implement ServerList­Filter(服务列表的过滤)
```


### 4. 超时时间
Ribbon 中有两种和时间相关的设置，分别是请求连接的超时时间和请求处理的超时时间，设置规则如下：
```
# 请求连接的超时时间
ribbon.ConnectTimeout=20000
# 请求处理的超时时间
ribbon.ReadTimeout=50000
```

也可以为每个Ribbon客户端设置不同的超时时间, 通过服务名称进行指定：
```
ribbon-config-demo.ribbon.ConnectTimeout=30000
ribbon-config-demo.ribbon.ReadTimeout=60000
```

### 5. 并发参数
```
# 最大连接数
ribbon.MaxTotalConnections=500
# 每个host最大连接数
ribbon.MaxConnectionsPerHost=500
```


## 重试机制
在集群环境中，用多个节点来提供服务，难免会有某个节点出现故障。用 Nginx 做负载均衡的时候，如果你的应用是无状态的、可以滚动发布的，也就是需要一台台去重启应用，这样对用户的影响其实是比较小的，因为 Nginx 在转发请求失败后会重新将该请求转发到别的实例上去。

由于 Eureka 是基于 AP 原则构建的，牺牲了数据的一致性，每个 Eureka 服务都会保存注册的服务信息，当注册的客户端与 Eureka 的心跳无法保持时，有可能是网络原因，也有可能是服务挂掉了。

在这种情况下，Eureka 中还会在一段时间内保存注册信息。这个时候客户端就有可能拿到已经挂掉了的服务信息，故 Ribbon 就有可能拿到已经失效了的服务信息，这样就会导致发生失败的请求。

这种问题我们可以利用重试机制来避免。重试机制就是当 Ribbon 发现请求的服务不可到达时，重新请求另外的服务。

### 1. RetryRule 重试
解决上述问题，最简单的方法就是利用 Ribbon 自带的重试策略进行重试，此时只需要指定某个服务的负载策略为重试策略即可：
```
ribbon-config-demo.ribbon.NFLoadBalancerRuleClassName=com.netflix.loadbalancer.RetryRule
```

### 2. Spring Retry 重试
除了使用 Ribbon 自带的重试策略，我们还可以通过集成 Spring Retry 来进行重试操作。

在 pom.xml 中添加 Spring Retry 的依赖，代码如下所示。
```
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
```

配置重试次数等信息：
```
# 对当前实例的重试次数
ribbon.maxAutoRetries=1
# 切换实例的重试次数
ribbon.maxAutoRetriesNextServer=3
# 对所有操作请求都进行重试
ribbon.okToRetryOnAllOperations=true
# 对Http响应码进行重试
ribbon.retryableStatusCodes=500,404,502
```
