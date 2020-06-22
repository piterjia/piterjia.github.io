---
layout: post
title: 【Spring Cloud系列】 Ribbon、Feign 以及 Hystrix 的关系
categories: 微服务
description: Spring Cloud 中 Ribbon、Feign 以及 Hystrix 的关系
keywords: Spring Cloud微服务熔断, Hystrix超时配置, Ribbon超时配置, Ribbon重试
---

在Spring Cloud中Ribbon、Feign、Hystrix，它们三者之间在处理微服务调用的关系是什么？

微服务之间的互相调用可以通过 Feign 进行声明式调用，在这个服务调用过程中 Feign 会通过 Ribbon 从服务注册中心获取目标服务的服务器地址列表，然后在网络请求的过程中 Ribbon 就会将请求以负载均衡的方式打到不同服务器上，从而实现 Spring Cloud 微服务架构中最关键的功能，即服务发现及客户端负载均衡调用。

微服务在互相调用的过程中，为了防止某个微服务的故障消耗掉整个系统所有微服务的连接资源，所以在微服务调用的过程中，我们会实施针对重点微服务的熔断逻辑。而要实现这个逻辑是通过 Hystrix 这个框架来实现的。

调用方会针对被调用微服务设置调用超时时间，一旦超时就会进入熔断逻辑，而这个故障指标信息也会返回给Hystrix组件，Hystrix组件会根据熔断情况判断被调微服务的故障情况从而打开熔断器，之后所有针对该微服务的请求就会直接进入熔断逻辑，直到被调微服务故障恢复，Hystrix断路器关闭为止。

三者的关系大致如下：

![](/images/microservice/hystrix-relation-1.png)


## Feign、Ribbon 以及 Hystrix 的配置说明

### Ribbon配置说明

Ribbon 的主要功能包括客户端负载均衡器及用于中间层通信的客户端。在基于Feign的微服务通信中无论是否开启Hystrix，Ribbon都是必不可少的，Ribbon的配置参数主要如下：

```
ribbon:
  #说明：同一台实例的最大自动重试次数，默认为1次，不包括首次
  MaxAutoRetries: 1
  #说明：要重试的下一个实例的最大数量，默认为1，不包括第一次被调用的实例
  MaxAutoRetriesNextServer: 1
  #说明：是否所有的操作都重试，默认为true
  OkToRetryOnAllOperations: true
  #说明：从注册中心刷新服务器列表信息的时间间隔，默认为2000毫秒，即2秒
  ServerListRefreshInterval: 2000
  #说明：使用Apache HttpClient连接超时时间，单位为毫秒
  ConnectTimeout: 3000
  #说明：使用Apache HttpClient读取的超时时间，单位为毫秒
  ReadTimeout: 3000
  #说明：初始服务器列表。一般不需要手工配置，在运行时会动态根据注册中心更新
  listOfServers: www.microsoft.com:80,www.yahoo.com:80,www.google.com:80
```

以上配置方式将对所有的微服务调用有效，如果想针对单独的微服务进行配置，使用“微服务名.ribbon”这样的配置方式即可，例如：

```
microserver1:
  ribbon:
    ReadTimeout: 30000
microserver1:
  ribbon:
    ReadTimeout: 30000
```


### Feign配置说明

Feign是一款Java语言编写的HttpClient绑定器，在Spring Cloud微服务中用于实现微服务之间的声明式调用，Feign自身可以支持多种HttpClient工具包，例如 OkHttp 及 Apache HttpClient。其默认常见配置如下：
```
feign:
  # 可选，替换掉JDK默认HttpURLConnection实现的 Http Client
  httpclient:
    enabled: true
  hystrix:
    enabled: true
  client:
    config:
      #JDK默认HttpURLConnection 实现的 Http Client配置
      default:
        #连接超时时间
        connectTimeout: 5000
        #读取超时时间
        readTimeout: 5000
        #错误解码器
        errorDecoder: com.wudimanong.client.common.FeignClientErrorDecoder
        #解码器
        encoder: com.wudimanong.client.common.FeignClientEncoder
        #编码器
        decoder: com.wudimanong.client.common.FeignClientDecoder
```

### Hystrix配置说明

Hystrix 主要被用于实现实现微服务之间网络调用故障的熔断、过载保护及资源隔离等功能。

接下来我们就按照参数配置的类型对 Hystrix 中的常见配置做一个梳理。

#### 1、线程隔离相关配置
Hystrix 具备的重要关键特性之一：它能够实现对第三方服务依赖的资源隔离，最常见的隔离方式是通过线程池资源的隔离来实现的，Hystrix 会为每个第三方服务依赖配置单独的线程池资源，从而避免对第三方服务依赖的请求占用应用主线程资源以免造成系统雪崩。

Hystrix中关于线程隔离相关的配置如下：

```
hystrix:
  command:
    #全局默认配置
    default:
      #线程隔离相关
      execution:
        timeout:
          #是否给方法执行设置超时时间，默认为true。一般我们不要改。
          enabled: true
        isolation:
          #配置请求隔离的方式，这里是默认的线程池方式。还有一种信号量的方式semaphore，使用比较少。
          strategy: threadPool
          thread:
            #方式执行的超时时间，默认为1000毫秒，在实际场景中需要根据情况设置
            timeoutInMilliseconds: 1000
            #发生超时时是否中断方法的执行，默认值为true。不要改。
            interruptOnTimeout: true
            #是否在方法执行被取消时中断方法，默认值为false。没有实际意义，默认就好！
            interruptOnCancel: false
```

#### 2、熔断器相关配置

熔断器是 Hystrix 最主要的功能，它开启和关闭的时机、灵敏度及准确性是 Hystrix 是否能够发挥重要的关键。

在 Hystrix 中与熔断器相关的几个配置如下：

```
hystrix:
  command:
    #全局默认配置
    default:
      #熔断器相关配置
      circuitBreaker:
        #说明：是否启动熔断器，默认为true。我们使用Hystrix的目的就是为了熔断器，不要改，否则就不要引入Hystrix。
        enabled: true
        #启用熔断器功能窗口时间内的最小请求数，假设我们设置的窗口时间为10秒，
        #那么如果此时默认值为20的话，那么即便10秒内有19个请求都失败也不会打开熔断器。
        #此配置项需要根据接口的QPS进行计算，值太小会有误打开熔断器的可能，而如果值太大超出了时间窗口内的总请求数，则熔断永远也不会被触发
        #建议设置一般为：QPS*窗口描述*60%
        requestVolumeThreshold: 20
        #熔断器被打开后，所有的请求都会被快速失败掉，但是何时恢复服务是一个问题。熔断器打开后，Hystrix会在经过一段时间后就放行一条请求
        #如果请求能够执行成功，则说明此时服务可能已经恢复了正常，那么熔断器会关闭；相反执行失败，则认为服务仍然不可用，熔断器保持打开。
        #所以此配置的作用是指定熔断器打开后多长时间内允许一次请求尝试执行，官方默认配置为5秒。
        sleepWindowInMilliseconds: 5000
        #该配置是指在通过滑动窗口获取到当前时间段内Hystrix方法执行失败的几率后，根据此配置来判断是否需要打开熔断器
        #这里官方的默认配置为50，即窗口时间内超过50%的请求失败后就会打开熔断器将后续请求快速失败掉
        errorThresholdPercentage: 50
        #说明：是否强制启用熔断器，默认false，不要动！
        forceOpen: false
        #说明：是否强制关闭熔断器，默认false，不要动！
        forceClosed: false
```


#### 3、Metrics（统计器）相关配置

Hystrix 是否正常工作，最主要的依赖就是根据捕获的调用指标信息来判断是否打开或者关闭熔断器，影响 Hystrix 行为很重要的因素就是以下 Hystrix 关于Metrics的配置。

Metrics 的配置中出现了两个比较频繁的概念：滑动窗口、桶，Hystrix的统计器就是由滑动窗口来实现的。

```
hystrix:
  command:
    #全局默认配置
    default:
      metrics:
        rollingStats:
          #此配置用于设置Hystrix统计滑动窗口的时间，单位为毫秒，默认设置为10000毫秒，即一个滑动窗口默认统计的是10秒内的请求数据。
          timeInMilliseconds: 10000
          #此属性指定了滑动统计窗口划分的桶数。默认为10。
          #需要注意的是，metrics.rollingStats.timeInMilliseconds % metrics.rollingStats.numBuckets == 0必须成立，否则就会抛出异常
          numBuckets: 10
 
        rollingPercentile:
          #此属性配置统计方法是否响应时间百分比，默认为true。
          #Hystrix会统计方法执行1%，10%，50%，90%，99%等比例请求的平均耗时用来生成统计图表。
          #如果禁用该参数设置false,那么所有汇总统计信息（平均值、百分位数）将返回-1。
          enabled: true
          #说明：统计响应时间百分比时的窗口大小，默认为60000毫秒，即1分钟
          timeInMilliseconds: 60000
          #此属性用于设置滑动百分比窗口要划分的桶数，默认为6。
          #需要注意的是，metrics.rollingPercentile.timeInMilliseconds % metrics.rollingPercentile.numBuckets == 0必须成立，否则会抛出异常
          numBuckets: 6
          #该属性表示统计响应时间百分比，每个滑动窗口的桶内要保存的请求数，默认为100。
          #即默认10秒的桶内，如果执行了500次请求，那么只有最后100次请求执行的信息会被保存到桶内。
          #增加这个值会增加内存消耗量，一般情况下无需更改。
          bucketSize: 100
 
        healthSnapshot:
          #该参数配置了健康数据统计器（会影响Hystrix熔断）中每个桶的大小，默认为500毫秒。
          #在统计时Hystrix通过metrics.rollingStats.timeInMilliseconds / metrics.healthSnapshot.intervalInMilliseconds计算出桶数。
          #在窗口滑动时，每滑过一个桶的时间就统计一次当前窗口内请求的失败率。
          intervalInMilliseconds: 500
```


#### 4、线程池相关配置

在前面提到过 Hystrix 实现对第三方服务依赖资源隔离最主要的方式就是通过线程池，而 Hystrix 内线程的使用是基于 Java 内置线程池的简单封装，通过以下 Hystrix 线程池参数我们可以控制执行 Hystrix 命令的线程池的行为。

具体如下：

```
hystrix:
  command:
    default:
      #说明：核心线程池的大小，默认值是10
      coreSize: 10
      #说明：是否允许线程池扩展到最大线程池数量，默认为false。
      allowMaximumSizeToDivergeFromCoreSize: false
      #说明：线程池中线程的最大数量，默认值是10。此配置项单独配置时并不会生效，需要启用allowMaximumSizeToDivergeFromCoreSize
      maximumSize: 10
      #作业队列的最大值，默认值为-1。表示队列会使用SynchronousQueue，此时值为0，Hystrix不会向队列内存放作业。
      #如果此值设置为一个正int型，队列会使用一个固定size的LinkedBlockingQueue，此时在核心线程池都忙碌的情况下，会将作业暂时存放在此队列内，但是超出此队列的请求依然会被拒绝
      maxQueueSize: -1
      #设置队列拒绝请求的阀值，默认为5。
      queueSizeRejectionThreshold: 5
      #控制线程在释放前未使用的时间，默认为1分钟。
      keepAliveTimeMinutes: 1
```


## Feign、Hystrix、Ribbon的超时配置关系

Feign、Hystrix、Ribbon 都有针对微服务超时的配置，而在开启熔断器功能后，这些超时配置会影响到熔断器及服务降级逻辑的行为，那么它们之间超时的配置有什么关系呢？

在Spring Cloud中使用 Feign 进行微服务调用分为两层：Ribbon 的调用及 Hystrix 的调用。所以 Feign 的超时时间就是 Ribbon 和 Hystrix 超时时间的结合，而如果不启用 Hystrix， 则 Ribbon 的超时时间就是 Feign 的超时时间配置，Feign 自身的配置会被覆盖。


而如果开启了 Hystrix，那么 Ribbon 的超时时间配置与 Hystrix 的超时时间配置则存在依赖关系，因为涉及到 Ribbon 的重试机制，所以一般情况下都是Ribbon 的超时时间小于 Hystrix 的超时时间，否则会出现以下错误：

```
2019-07-12 11:10:20,238 397194 [http-nio-8084-exec-2] WARN  o.s.c.n.z.f.r.s.AbstractRibbonCommand - The Hystrix timeout of 40000ms for the command operation is set lower than the combination of the Ribbon read and connect timeout, 80000ms.
```

Ribbon 和 Hystrix 的超时时间配置的关系如下：
```
Hystrix的超时时间=Ribbon的重试次数(包含首次)*(ribbon.ReadTimeout+ribbon.ConnectTimeout)
```

而Ribbon的重试次数的计算方式为：
```
Ribbon重试次数(包含首次)=1+ribbon.MaxAutoRetries+ribbon.MaxAutoRetriesNextServer+(ribbon.MaxAutoRetries*ribbon.MaxAutoRetriesNextServer)
```


以我们文中的 Ribbon 配置为例子， Ribbon 的重试次数=1+(1+1+1)*(30000+10000)，所以 Hystrix 的超时配置应该>=160000毫秒。在 Ribbon 超时但 Hystrix 没有超时的情况下，Ribbon 便会采取重试机制；而重试期间如果时间超过了 Hystrix 的超时配置则会立即被熔断（fallback）。

如果不配置 Ribbon 的重试次数，则 Ribbon 默认会重试一次，加上第一次调用 Ribbon 的重试次数为2次，以上述配置为例 Hystrix 超时时间配置为2*40000=80000，大家一般不会主动配置Ribbon的重试次数。

以上超时配置的值只是示范，大家根据实际情况设置即可！


参考资料：
https://github.com/Netflix/Hystrix/wiki/Configuration
https://www.cnblogs.com/throwable/p/11961016.html



