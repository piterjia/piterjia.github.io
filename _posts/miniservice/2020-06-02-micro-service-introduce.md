---
layout: post
title: 【Spring Cloud系列】大话 Spring Cloud 微服务系统架构
categories: 微服务
description: Spring Cloud 微服务系统架构科普
keywords: spring Cloud, distributed, micro service
---

Spring Cloud 就是微服务系统架构的一站式解决方案，在平时我们构建微服务的过程中需要做如 **服务发现注册 、配置中心 、消息总线 、负载均衡 、断路器 、数据监控** 等操作，在微服务化过程中碰到的任何问题，都可以从Spring全家桶里找到现成的解决方案，而且方案还不止一种。

## Spring Cloud 的版本

Spring Cloud 的版本号并不是我们通常见的数字版本号，而是以字母表的顺序来对应不同的版本，比如：最早 的 Release 版本 Angel，第二个 Release 版本 Brixton，然后是 Camden、 Dalston、Edgware、Finchley、Greenwich、Hoxton。




## Spring Cloud的全景图
我们先来看一幅Spring Cloud的全景图，然后再详细解释其中的每个组件的功能。

![](/images/microservice/microserviceintroduce.png)




## 服务发现与治理框架——Eureka

在Spring Cloud的架构中，服务治理是其中不可或缺的核心环节，它包含了服务从注册到销毁的整个生命周期的管理。用一句话来说，服务治理确保了调用方可以准确的向可用的服务节点发起调用。

Spring Cloud提供了三款服务治理的组件，分别是 Eureka, Consul 和 Nacus。现在用的比较多的是 Eureka。

### 举一个生活中的例子，租房子找中介。

在没有中介的时候，我们需要自己一个一个去寻找有房屋要出租的房东，这显然会非常的费力。凭一个人的能力找不到很多房源，再者也很低效。这里的我们就**相当于微服务中的 Consumer ，而那些房东就相当于微服务中的 Provider 。消费者 Consumer 需要调用提供者 Provider 提供的一些服务，就像我们现在需要租他们的房子一样。**

房东找不到租客，租客找不到房东。所以，房东肯定想到要广播自己的房源信息，让更多的租客获悉他的房源信息。

但是有两个问题就出现了。
- 第一、其他不是租客的也收到了广播的租房消息，在计算机的世界就会出现资源消耗的问题。
- 第二、租客这样还是很难主动找到你，没有一个汇总统一房源的地方。

那怎么办呢？这就是中介的作用，它为我们提供了统一房源的地方，我们租客只需要去它那里找就行。对于房东，他们也只需要把房源在中介那里发布就行。

那么现在，我们的模式就是这样的了。
![](/images/microservice/房源中介.png)


### 例子扩展
但是，这个时候还会出现一些问题。
- 房东注册之后如果不想卖房子了怎么办？我们是不是需要让房东定期续约？如果房东不进行续约是不是要将他们从中介那里的注册列表中移除？
- 租客是不是也要进行注册呢？不注册怎么拿到房源信息呢？
- 中介可不可以做连锁店呢？如果这一个店因为某些不可抗力而无法使用，那么我们是否可以换一个连锁店呢？
  
针对上面的问题我们来重新构建一下上面的模式图
![](/images/microservice/房源中介2.jpg)



### Eureka 基础概念介绍
下面我们来看关于 Eureka 的一些基础概念。

#### 概念1：服务发现：
其实就是一个“中介”，整个过程中有三个角色：服务提供者(出租房子的)、服务消费者(租客)、服务中介(房屋中介)。

- 服务提供者： 就是提供一些自己的服务给外界。
- 服务消费者： 就是需要使用一些服务的“用户”。
- 服务中介： 服务提供者和服务消费者之间的“桥梁”。
  
服务提供者可以把自己注册到服务中介那里，而服务消费者如需要消费一些服务。就可以在服务中介中寻找注册在服务中介的服务提供者。

#### 概念2：服务注册：

官方解释：
- 当 Eureka 客户端向 Eureka Server 注册时，它提供自身的元数据，比如IP地址、端口，运行状况指示符URL，主页等。

结合中介理解：
- 房东 (提供者 Eureka Client Provider)在中介 (服务器 Eureka Server) 那里登记房屋的信息，比如面积，价格，地段等等(元数据 metaData)。
- 租客 (提供者 Eureka Client Consumer)在中介 (服务器 Eureka Server) 那里登记他的意向房屋信息，比如面积，价格，地段等等(元数据 metaData)。

#### 概念3：服务续约：
官方解释：
- Eureka 客户会每隔30秒(默认情况下)发送一次心跳来续约。 通过续约来告知 Eureka Server 该 Eureka 客户仍然存在，没有出现问题。 正常情况下，如果 Eureka Server 在90秒没有收到 Eureka 客户的续约，它会将实例从其注册表中删除。

结合中介理解：
- 房东 (提供者 Eureka Client Provider) 定期告诉中介 (服务器 Eureka Server) 我的房子还租(续约) ，中介 (服务器Eureka Server) 收到之后继续保留房屋的信息。

#### 概念4：获取注册列表信息：

官方解释：
- Eureka 客户端从服务器获取注册表信息，并将其缓存在本地。客户端会使用该信息查找其他服务，从而进行远程调用。
- 该注册列表信息定期（每30秒钟）更新一次。每次返回的注册列表信息可能与 Eureka 客户端的缓存信息不同, Eureka 客户端自动处理。
- 如果由于某种原因导致注册列表信息不能及时匹配，Eureka 客户端则会重新获取整个注册表信息。 
- Eureka 服务器缓存注册列表信息，整个注册表以及每个应用程序的信息进行了压缩，压缩内容和没有压缩的内容完全相同。
- Eureka 客户端和 Eureka 服务器可以使用 JSON / XML 格式进行通讯。默认的情况下 Eureka 客户端使用压缩 JSON 格式来获取注册列表的信息。

结合中介理解：
- 租客(消费者 Eureka Client Consumer) 去中介 (服务器 Eureka Server) 那里获取所有的房屋信息列表 (客户端列表 Eureka Client List) 
- 租客为了获取最新的信息会定期向中介 (服务器 Eureka Server) 那里获取并更新本地列表。

#### 概念5：服务下线：

官方解释：
- Eureka客户端在程序关闭时，向Eureka服务器发送取消请求。 
- 发送请求后，该客户端实例信息将从服务器的实例注册表中删除。该下线请求不会自动完成，它需要调用以下内容：DiscoveryManager.getInstance().shutdownComponent();

结合中介理解：
- 房东 (提供者 Eureka Client Provider) 告诉中介 (服务器 Eureka Server) 我的房子不租了，中介之后就将注册的房屋信息从列表中剔除。


#### 概念6：服务剔除：

官方解释：
- 在默认的情况下，当 Eureka 客户端连续90秒(3个续约周期)没有向 Eureka 服务器发送服务续约，即心跳，Eureka 服务器会将该服务实例从服务注册列表删除，即服务剔除。

结合中介理解：
- 房东(提供者 Eureka Client Provider) 会定期联系中介 (服务器 Eureka Server) 告诉他我的房子还租，如果中介 (服务器 Eureka Server) 长时间没收到提供者的信息，那么中介会将他的房屋信息给下架。

**下面就是 Netflix 官方给出的 Eureka 架构图：**

![](/images/microservice/eureka%20server.png)






## 负载均衡
Ribbon是Spring Cloud中负责负载均衡的组件，Ribbon的一大优势是它能够和各个Spring Cloud组件无缝集成，而且十分灵巧轻便又具备高可扩展性。

负载均衡框架是起到分散服务器压力的作用，可以这么说，没有负载均衡技术的服务器集群就不能叫做集群，只有借助负载均衡技术，集群才能够借助服务节点的规模效应发挥出优势。


### 什么是 RestTemplate?
RestTemplate 是 Spring 提供的一个访问 Http 服务的客户端类，怎么说呢？就是微服务之间的调用是使用的 RestTemplate。比如这个时候我们消费者B需要调用提供者A所提供的服务我们就需要这么写。

如下面的伪代码:
```
@Autowired
private RestTemplate restTemplate;
// 这里是提供者A的ip地址，但是如果使用了 Eureka 那么就应该是提供者A的名称
private static final String SERVICE_PROVIDER_A = "http://localhost:8081";

@PostMapping("/judge")
public boolean judge(@RequestBody Request request) {
    String url = SERVICE_PROVIDER_A + "/service1";
    return restTemplate.postForObject(url, request, Boolean.class);
}
```

### 为什么需要 Ribbon？

Ribbon 是 Netflix 公司的一个开源的负载均衡项目，是**一个客户端/进程内负载均衡器，运行在消费者端。**

我们再举一个秒杀系统的例子，为了整个系统的高可用，我们需要将这个系统做一个集群，消费者就可以拥有多个秒杀系统的调用途径了，如下图:

![](/images/microservice/秒杀系统-ribbon.jpg)

如果这个时候没有进行一些**均衡操作**，比如我们对秒杀系统1进行大量的调用，而另外两个基本不请求，就会导致秒杀系统1崩溃，而另外两个就变成了傀儡，那么我们为什么还要做集群，我们高可用体现的意义又在哪呢？

注意：Ribbon 是运行在消费者端的负载均衡器，如下图：
![](/images/microservice/秒杀系统-ribbon2.jpg)

其工作原理就是 Consumer 端获取到了所有的服务列表之后，在其内部使用负载均衡算法，再展开对多个系统调用。


### Nginx 和 Ribbon 的对比
提到 **负载均衡** 就不得不提到大名鼎鼎的 Nignx 了，而和 Ribbon 不同的是，它是一种集中式的负载均衡器。

简单理解就是，Nginx 在请求入口处，将所有请求都集中起来，然后再进行负载均衡。如下图。

![](/images/microservice/nginx-vs-ribbon1.jpg)


我们可以看到 Nginx 是接收了所有的请求进行负载均衡的，而对于 Ribbon 来说它是在消费者端进行的负载均衡。如下图:

![](/images/microservice/nginx-vs-ribbon2.jpg)

```
在 Nginx 中请求是先进入负载均衡器，而在 Ribbon 中是先在客户端进行负载均衡才进行请求的。
```

### Ribbon 的几种负载均衡算法
Ribbon 中默认使用的 RoundRobinRule 轮询策略。
- RoundRobinRule：轮询策略。Ribbon 默认采用的策略。若经过一轮轮询没有找到可用的 provider，其最多轮询 10 轮。若最终还没有找到，则返回 null。
- RandomRule: 随机策略，从所有可用的 provider 中随机选择一个。
- RetryRule: 重试策略。先按照 RoundRobinRule 策略获取 provider，若获取失败，则在指定的时限内重试。默认的时限为 500 毫秒。

如果要更换默认的负载均衡算法，只需要在配置文件中做出修改就行：
```
providerName:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
```



## 消息间调用
对于原生态的HTTP调用来说，从Java代码里发起调用并且构造消息体和Header是一件非常麻烦的事情，考虑到Spring Cloud的服务治理组件也是基于 HTTP 的，因此特别需要一款简化服务调用的组件。Feign 的出现就是为了解决这个问题，我们可以借助 Feign 的代理机制，像调用一个接口方法一样发起远程 HTTP 调用。

有了 Eureka ，RestTemplate ，Ribbon，如果使用 RestTemplate 每次都要进行这样的调用。

```
@Autowired
private RestTemplate restTemplate;
// 这里是提供者A的ip地址，但是如果使用了 Eureka 那么就应该是提供者A的名称
private static final String SERVICE_PROVIDER_A = "http://localhost:8081";

@PostMapping("/judge")
public boolean judge(@RequestBody Request request) {
    String url = SERVICE_PROVIDER_A + "/service1";
    // 是不是太麻烦了？？？每次都要 url、请求、返回类型的 
    return restTemplate.postForObject(url, request, Boolean.class);
}
```

这样每次都调用 RestRemplate 的 API 是否太麻烦，那有没有更简单的方式呢？ Feign ！！ 
OpenFeign 也是运行在消费者端，使用 Ribbon 进行负载均衡，所以 OpenFeign 直接内置了 Ribbon。

在导入了 Open Feign 之后我们就可以进行愉快编写 Consumer 端代码了。
```
// 使用 @FeignClient 注解来指定提供者的名字
@FeignClient(value = "eureka-client-provider")
public interface TestClient {
    // 这里一定要注意需要使用的是提供者那端的请求相对路径，这里就相当于映射了
    @RequestMapping(value = "/provider/xxx", method = RequestMethod.POST)
    CommonResponse<List<Plan>> getPlans(@RequestBody planGetRequest request);
}
```

然后我们在 Controller 就可以像原来调用 Service 层代码一样调用它了。
```
@RestController
public class TestController {
    // 这里就相当于原来自动注入的 Service
    @Autowired
    private TestClient testClient;
    // controller 调用 service 层代码
    @RequestMapping(value = "/test", method = RequestMethod.POST)
    public CommonResponse<List<Plan>> get(@RequestBody planGetRequest request) {
        return testClient.getPlans(request);
    }
}
```



## 服务容错
Hystrix 是目前 Spring Cloud 中应用最广泛的服务容错组件.

服务容错就是尽可能降低服务异常所带来的影响。我们经常听到两个词叫做 **“降级” 和 “熔断”** 。
- 降级，很好理解，降级就是退而求其次，在异常发生之后选一种备选方案继续提供服务。
- 熔断，是指在异常达到某个临界值以后，直接切断服务通路，将用户请求统统导向降级逻辑中。

举个例子：比如我们的微服务系统是这样的。服务A调用了服务B，服务B再调用了服务C，但是因为某些原因，服务C顶不住了，这个时候大量请求会在服务C阻塞。

![](/images/microservice/Hystrix1.jpg)

这个时候因为服务C不能返回响应，那么服务B调用服务C的的请求就会阻塞，同理服务B阻塞了，那么服务A也会阻塞。于是，发生了服务雪崩。

![](/images/microservice/Hystrix2.jpg)

**熔断就是服务雪崩的一种有效解决方案。当指定时间窗内的请求失败率达到设定阈值时，系统会直接将此请求链路断开。**
也就是我们上面服务B调用服务C在指定时间窗内，调用的失败率到达了一定的值，那么 Hystrix 则会自动将服务B与C之间的请求都断了，以免导致服务雪崩现象。

**降级是为了更好的用户体验，当一个方法调用异常时，通过执行另一种代码逻辑来给用户友好的回复。这也就对应着 Hystrix 的后备处理模式。** 





## 分布式配置中心和消息推送组件
Spring Cloud 借助 Config 组件来集中管理集群中所有服务节点的配置，它是一个中心化的配置管理中心，将你的微服务从繁重的配置工作中解脱出来。利用 Config 组件我们可以轻松玩转环境隔离、配置推送和配置项动态刷新。

提到配置属性的刷新，就不得不说到 Spring Cloud 中的另一个组件 Bus，它承担了批量通知和推送配置变更的工作，而且我们可以通过扩展 Bus 的事件，实现“消息广播”的应用场景。

### Config 是什么
简单来说，Spring Cloud Config 就是能将各个 应用/系统/模块 的配置文件存放到统一的地方然后进行管理(Git 或者 SVN)。

我们的应用启动的时候会进行配置文件的加载，那么我们的 Spring Cloud Config 就暴露出一个接口给启动应用来获取它所想要的配置文件，应用获取到配置文件，然后再进行它的初始化工作。就如下图。

![](/images/microservice/config-ksksks.jpg)

### 组件 Bus

在应用运行时去更改远程配置仓库(Git)中的对应配置文件，依赖这个配置文件的应用如何马上进行配置更新？我们会使用 Bus 消息总线 + Spring Cloud Config 进行配置的动态刷新。

简单理解 Spring Cloud Bus 的作用就是管理和广播分布式系统中的消息，也就是消息引擎系统中的广播模式。当然作为消息总线 的 Spring Cloud Bus 可以做很多事，而不仅仅是客户端的配置刷新功能。

而拥有了 Spring Cloud Bus 之后，我们只需要创建一个简单的请求，并且加上 @ResfreshScope 注解就能进行配置的动态修改了。

如图所示。
![](/images/microservice/springcloud-bus-s213dsfsd.jpg)






## 服务网关
服务网关是微服务的第一道关卡，目前 Nginx 是应用最广泛的反向代理技术，在各个大厂的核心业务系统中都有大量应用，不过 Nginx 可不是使用 Java 来配置的，使用和配置 Nginx 需要掌握它的语法树。

Spring Cloud 则为广大的 Java 技术人员提供了更加“编程友好”的方式来构建网关层，那就是 Gateway 和 Zuul 网关层组件。我们可以通过 Java 代码或者是 yml 配置文件的方式编写自己的路由规则，并通过内置过滤器或自定义过滤器来实现复杂的业务需求。Gateway 本身也集成了强大的限流功能，结合使用 Redis+SpEL 表达式，可以对业务系统进行精准限流。

### Zuul
我们知道 服务提供者 是 消费者 通过 Eureka Server 进行访问的，即 Eureka Server 是 服务提供者 的统一入口。那么整个应用中存在那么多消费者需要用户进行调用，这个时候用户该怎样访问这些消费者工程呢？当然可以像之前那样直接访问这些工程。但这种方式没有统一的消费者工程调用入口，不便于访问与管理，而 Zuul 就是这样的一个对于 消费者 的统一入口。

简单来讲网关是系统唯一对外的入口，介于客户端与服务器端之间，用于对请求进行鉴权、限流、 路由、监控等功能。

![](/images/microservice/zuul-sj22o93nfdsjkdsf.jpg)


Zuul 中最关键的就是**路由和过滤器**

### Zuul 的路由功能

举个例子：比如这个时候我们已经向 Eureka Server 注册了两个 Consumer 、三个 Provicer ，这个时候我们再加个 Zuul 网关应该变成这样子了。
![](/images/microservice/zuul-sj22o93nfdsjkdsf2312.jpg)

- 1）首先，Zuul 需要向 Eureka 进行注册。
- 2）Consumer 已经向 Eureka Server 进行注册了，网关注册之后就能拿到所有 Consumer 的信息。并获取到所有的 Consumer 的元数据(名称，ip，端口)
- 3）拿到这些元数据之后，我们就可以直接可以做路由映射了。比如原来用户调用 Consumer1 的接口 localhost:8001/studentInfo/update 这个请求，我们是不是可以这样进行调用了呢？localhost:9000/consumer1/studentInfo/update 呢？

### Zuul 的过滤功能
毕竟所有请求都经过网关(Zuul)，那么我们可以进行各种过滤，这样我们就能实现**限流，灰度发布，权限控制**等.

要实现自己定义的 Filter 我们只需要继承 ZuulFilter 然后将这个过滤器类以 @Component 注解加入 Spring 容器中就行了。


### 关于 Zuul 的其他
Zuul 的过滤器的功能还可以实现 权限校验，灰度发布、限流等等。细节我们后面用一整篇文章单独说。

当然，Zuul 作为网关肯定也存在**单点问题**，如果我们要保证 Zuul 的高可用，我们就需要进行 Zuul 的集群配置，这个时候可以借助额外的一些负载均衡器比如 Nginx 。






## 调用链路追踪
微服务的一大特点就是完成一个业务场景所需要调用的上下游链路非常长，比如说一个下单操作，后台就要调用商品、订单、营销优惠、履约、消息推送、支付等等一大家子微服务，任何一个环节出错可能都会导致下单失败。

Sleuth 是 Spring Cloud 提供的调用链路追踪组件，它进行线上问题排查必不可少的关键环节，单就 Sleuth 来说，它就是在一整条调用链路中打上某个标记，将一个 api 请求所调用的所有上下游链路串联起来。如果从宏观的角度来说，调用链追踪还涉及到日志打标、调用链分析、日志收集、构建搜索 Index 等等流程。





## 消息驱动
Kafka 和 RabbitMQ 是目前应用最广泛的消息中间件，很多异步调用场景底层都依赖于消息组件，比如说电商场景中的商品批量发布，或者下单成功后的邮件通知系统等等。Stream 是Spring Cloud 为我们提供的消息驱动组件，它代理了业务层和底层的物理中间件的交互，至于底层中间件是 Kafka 还是 RabbitMQ ，对业务层几乎是无感知的。借助Stream我们不仅可以轻松实现组内单播和广播场景，同时Stream还提供了对异常处理的丰富支持。





## 防流量卫兵
Sentinel 是阿里巴巴开源的一款主打“流量控制”的组件，与Spring Cloud、Dubbo、甚至 GRPC 都可以很好的集成，在分布式流量控制（包括秒杀场景的突发流量场景）、熔断、消息驱动下的削峰填谷等各个场景下都有稳定发挥。






## 小结
本文我们对Spring的整体架构做了一个简单的讲解，大家对 Spring Cloud 有一个朦胧的概念，大致了解下 Spring Cloud 都有哪些组件，后面我们会针对每一种组件做一个详细的介绍。

![](/images/microservice/微服务三大派系.png)

从上面的表格中可以看出，在大多数的领域当中，都有多于一种的解决方案，而且各个组织在不同领域发力程度也不一样。我们在实际的研发一般会结合使用来自不同组织的组件，这样才能发挥 Spring Cloud 的最大优势。

