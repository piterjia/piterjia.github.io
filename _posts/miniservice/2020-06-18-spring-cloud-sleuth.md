---
layout: post
title: 【Spring Cloud 系列】 服务链路追踪 Sleuth
categories: 微服务
description: 服务链路追踪 Sleuth
keywords: Spring Cloud, Sleuth, 链路追踪
---

Spring Cloud Sleuth 是分布式系统中跟踪服务间调用的工具，它可以直观地展示出一次请求的调用过程，本文将对其用法进行详细介绍。

## Spring Cloud Sleuth 简介
随着我们的系统越来越庞大，各个服务间的调用关系也变得越来越复杂。当客户端发起一个请求时，这个请求经过多个服务后，最终返回了结果，经过的每一个服务都有可能发生延迟或错误，从而导致请求失败。这时候我们就需要请求链路跟踪工具来帮助我们，理清请求调用的服务链路，解决问题。


## 基本术语

- Span：基本工作单元，发送一个远程调度任务 就会产生一个Span，Span是一个64位ID唯一标识的，Trace是用另一个64位ID唯一标识的，Span还有其他数据信息，比如摘要、时间戳事件、Span的ID、以及进度ID。
- Trace：一系列Span组成的一个树状结构。请求一个微服务系统的API接口，这个API接口，需要调用多个微服务，调用每个微服务都会产生一个新的Span，所有由这个请求产生的Span组成了这个Trace。
- Annotation：用来及时记录一个事件的，一些核心注解用来定义一个请求的开始和结束 。这些注解包括以下：
  - cs ： Client Sent -客户端发送一个请求，这个注解描述了这个Span的开始
  - sr - Server Received -服务端获得请求并准备开始处理它，如果将其sr减去cs时间戳便可得到网络传输的时间。
  - ss - Server Sent （服务端发送响应）–该注解表明请求处理的完成(当请求返回客户端)，如果ss的时间戳减去sr时间戳，就可以得到服务器请求的时间。
  - cr - Client Received （客户端接收响应）-此时Span的结束，如果cr的时间戳减去cs时间戳便可以得到整个请求所消耗的时间。


## 案例实战

本文案例一共四个工程，采用多个子 Module 形式。需要新建一个主 Maven 工程。包含了：
- eureka-server 工程，作为服务注册中心，eureka-server 的创建过程这里不重复；参考我之前的文章 [Eureka 单节点注册中心](2020-06-04-micro-service-eureka.md)
- zipkin-server 作为链路追踪服务中心，负责存储链路数据；
- sleuth-traceA 为一个应用服务，对外暴露API接口，同时它也作为链路追踪客户端，负责产生数据。并上传给zipkin-service；
- sleuth-traceA 为一个应用服务，对外暴露API接口，同时它也作为链路追踪客户端，负责产生数据。


### 构建 zipkin-server 工程

新建一个 Module 工程，取名为 zipkin-server，其pom文件继承了主 Maven 工程的 pom 文件；作为Eureka Client，引入 Eureka 的依赖 spring-cloud-starter-netflix-eureka-client，引入zipkin-server 依赖，以及 zipkin-autoconfigure-ui 依赖，后两个依赖提供了 Zipkin 的功能和Zipkin 界面展示的功能。代码如下：

#### zipkin-server pom文件的依赖关系
```
<dependencies>
    <dependency>
        <groupId>io.zipkin.java</groupId>
        <artifactId>zipkin-server</artifactId>
        <version>2.8.4</version>
    </dependency>

    <dependency>
        <groupId>io.zipkin.java</groupId>
        <artifactId>zipkin-autoconfigure-ui</artifactId>
        <version>2.8.4</version>
    </dependency>

    <!-- 高可用 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
</dependencies>
```

#### zipkin-server 启动类

在程序的启动类 ZipkinServiceApplication 加上 @EnableZipkinServer ，开启 ZipkinServer 的功能，加上 @EnableEurekaClient 注解，启动 Eureka Client。代码如下：

```
@SpringBootApplication
@EnableZipkinServer
public class ZipkinApplication {

    public static void main(String[] args) {
        new SpringApplicationBuilder(ZipkinApplication.class)
                .web(WebApplicationType.SERVLET)
                .run(args);
    }
}
```

#### zipkin-server 配置文件

在配置文件 application.properties 文件，指定程序名为 zipkin-server，端口为 62100 ，服务注册地址为 http://localhost:20000/eureka/。

```
spring.application.name=zipkin-server
server.port=62100

spring.main.allow-bean-definition-overriding=true

eureka.client.serviceUrl.defaultZone=http://localhost:20000/eureka/

management.metrics.web.server.auto-time-requests=false

```


### 构建 sleuth-traceB

在主 Maven 工程下建一个 Module 工程，取名为 sleuth-traceB，作为应用服务，对外暴露 API 接口。pom 文件继承了主Maven 工程的 pom 文件，并引入了 Eureka 的起步依赖 spring-cloud-starter-eureka，Web 起步依赖spring-boot-starter-web，Zipkin 的起步依赖 spring-cloud-starter-zipkin，代码如下：

#### sleuth-traceB pom文件的依赖关系
```
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- 链路追踪 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-sleuth</artifactId>
    </dependency>
    <!-- Zipkin -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-zipkin</artifactId>
    </dependency>
</dependencies>
```


#### sleuth-traceB 配置文件

在配置文件applicatiom.properties，指定了程序名为 sleuth-traceB，端口为62001，服务注册地址为 http://localhost:20000/eureka/，Zipkin Server 地址为 http://localhost:62100。spring.sleuth.sampler.probability 为 1.0，即 100% 的概率将链路的数据上传给 Zipkin Server，代码如下：

```
spring.application.name=sleuth-traceB
server.port=62001

eureka.client.serviceUrl.defaultZone=http://localhost:20000/eureka/

logging.file=${spring.application.name}.log


# zipkin的地址
spring.zipkin.base-url=http://localhost:62100

info.app.name=sleuth-traceB
info.app.description=test

spring.sleuth.sampler.probability=1

management.security.enabled=false
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always

```



#### sleuth-traceB 启动类

在程序的启动类 ZipkinServiceApplication 加上 @EnableDiscoveryClient ，启动 Eureka Client。

为了方便，直接在启动类上添加 @RestController 注解，启动 作为 controller 提供服务。代码如下：

```
@EnableDiscoveryClient
@SpringBootApplication
@RestController
@Slf4j
public class SleuthTraceBMain {

    @LoadBalanced
    @Bean
    public RestTemplate lb() {
        return new RestTemplate();
    }

    @Autowired
    private RestTemplate restTemplate;


    @GetMapping(value = "/traceB")
    public String traceB() {
        log.info("-------Trace B");
        return "traceB";
    }


    public static void main(String[] args) {
        new SpringApplicationBuilder(SleuthTraceBMain.class)
                .web(WebApplicationType.SERVLET)
                .run(args);
    }

}
```


### 构建 sleuth-traceA

在主 Maven 工程下建一个 Module 工程，取名为 sleuth-traceA，作为应用服务，对外暴露 API 接口。pom 文件继承了主Maven 工程的 pom 文件，并引入了 Eureka 的起步依赖 spring-cloud-starter-eureka，Web 起步依赖spring-boot-starter-web，Zipkin 的起步依赖 spring-cloud-starter-zipkin，代码如下：

#### sleuth-traceB pom文件的依赖关系
```
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- 链路追踪 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-sleuth</artifactId>
    </dependency>
    <!-- Zipkin -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-zipkin</artifactId>
    </dependency>
</dependencies>
```


#### sleuth-traceA 配置文件

在配置文件applicatiom.properties，指定了程序名为 sleuth-traceA，端口为62000，服务注册地址为 http://localhost:20000/eureka/，Zipkin Server 地址为 http://localhost:62100。spring.sleuth.sampler.probability 为 1.0，即 100% 的概率将链路的数据上传给 Zipkin Server，代码如下：

```
spring.application.name=sleuth-traceA
server.port=62000

eureka.client.serviceUrl.defaultZone=http://localhost:20000/eureka/

logging.file=${spring.application.name}.log

# zipkin的地址
spring.zipkin.base-url=http://localhost:62100
spring.sleuth.sampler.probability=1

info.app.name=sleuth-traceA
info.app.description=test

management.security.enabled=false
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always

```



#### sleuth-traceA 启动类

在程序的启动类 ZipkinServiceApplication 加上 @EnableDiscoveryClient ，启动 Eureka Client。

为了方便，直接在启动类上添加 @RestController 注解，启动 作为 controller 提供服务。并通过代码服务接口traceA 实现对 服务B 的调用。代码如下：

```
@EnableDiscoveryClient
@SpringBootApplication
@RestController
@Slf4j
public class SleuthTraceAMain {

    @LoadBalanced
    @Bean
    public RestTemplate lb() {
        return new RestTemplate();
    }

    @Autowired
    private RestTemplate restTemplate;


    @GetMapping(value = "/traceA")
    public String traceA() {
        log.info("-------Trace A");
        return restTemplate.getForEntity("http://sleuth-traceB/traceB", String.class)
                .getBody();
    }


    public static void main(String[] args) {
        new SpringApplicationBuilder(SleuthTraceAMain.class)
                .web(WebApplicationType.SERVLET)
                .run(args);
    }
}
```


## 项目演示

完整的项目搭建完毕，依次启动 eureka-server、zipkin-server、sleuth-traceB、sleuth-traceA。使用postman 访问http://localhost:62000/traceA，结果如下

![](/images/microservice/sleuth1.png)

访问 http://localhost:62100，即访问 Zipkin 的展示界面，界面显示如图所示：


![](/images/microservice/sleuth3.png)


这个界面主要用来查找服务的调用情况，可以根据服务名、开始时间、结束时间、请求消耗的时间等条件来查找。点击 “Find Trackes” 按钮，界面如图所示。从图可知服务的调用情况，比如服务调用时间、服务的消耗时间，服务调用的链路情况。

![](/images/microservice/sleuth2.png)


点击其中一个调用链，进入查看详细的调用关系：

![](/images/microservice/sleuth5.png)


点击Dependences按钮，可以查看服务的依赖关系，在本案例中，它们的依赖关系如图：

![](/images/microservice/sleuth4.png)



## 在Sleuth链路中添加自定义 Tag 标签
通常我们会在链路日志中添加额外的自定义字段，帮助我们进行链路分析。我们可以借助brave.Tracer类实现这一目标。

修改 slenth traceA 的代码,


### 首先在代码中注入Tracer类：

```
private Tracer tracer;

@Autowired
public void setTracer(Tracer tracer) {
	this.tracer = tracer;
}
```

### 然后将我们指定的字段添加到当前Span中：
```
tracer.currentSpan().tag("transId", "11111");
tracer.currentSpan().tag("appId", "22222");
tracer.currentSpan().tag("reqTime", LocalDateTime.now().toString());
```


### 整体的代码如下：

```
@EnableDiscoveryClient
@SpringBootApplication
@RestController
@Slf4j
public class SleuthTraceAMain {

    @LoadBalanced
    @Bean
    public RestTemplate lb() {
        return new RestTemplate();
    }

    @Autowired
    private Tracer tracer;


    @Autowired
    private RestTemplate restTemplate;

    @GetMapping(value = "/traceA")
    public String traceA() {
        log.info("-------Trace A");
        return restTemplate.getForEntity("http://sleuth-traceB/traceB", String.class)
                .getBody();
    }

    @GetMapping(value = "/traceAWithTag")
    public String traceAWithTag() {
        log.info("-------trace A With Tag");
        tracer.currentSpan().tag("transId", "11111");
        tracer.currentSpan().tag("appId", "22222");
        tracer.currentSpan().tag("reqTime", LocalDateTime.now().toString());
        return restTemplate.getForEntity("http://sleuth-traceB/traceB", String.class)
                .getBody();
    }


    public static void main(String[] args) {
        new SpringApplicationBuilder(SleuthTraceAMain.class)
                .web(WebApplicationType.SERVLET)
                .run(args);
    }

}
```


### 访问Zipkin服务端可以看到，Tags列中已经包含我们添加的字段。

![](/images/microservice/sleuth6.png)





